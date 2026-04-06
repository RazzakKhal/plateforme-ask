# Specifications fonctionnelles et techniques

## 1. Contexte actuel

Le projet est compose de plusieurs microservices Spring Boot et d'un frontend Angular.

Constats observes dans l'etat actuel du depot :

- La deconnexion existe seulement cote frontend via suppression du token dans `localStorage`.
- Il n'existe pas d'endpoint backend de deconnexion.
- Les formulaires frontend ont surtout des validations `required`, avec peu de validations de format ou de coherence.
- Les DTO backend ne portent quasiment aucune validation Bean Validation.
- Les controllers n'utilisent pas `@Valid` sur les `@RequestBody`.
- La gestion d'erreur est heterogene :
    - `user-service` a quelques exceptions metier et un `GlobalExceptionHandlerController`.
    - `plateforme-formula-service` intercepte `Exception.class` et renvoie toujours `400`.
    - `plateforme-payment-service` et `plateforme-notification-service` n'ont pas de handler global equivalent.
- Le frontend suppose des erreurs au format `{ message, status }`, sans contrat commun partage.

L'objectif de ce document est de definir un cadre commun pour :

- la deconnexion front et back ;
- la validation des formulaires frontend et des DTO backend ;
- la standardisation des erreurs via `ApiException` et un contrat unique entre microservices.

## 2. Specs de deconnexion

### 2.1 Objectif

Ajouter une deconnexion coherente sur toute la plateforme :

- cote frontend : vider proprement la session locale et remettre l'application dans un etat non authentifie ;
- cote backend : invalider le JWT courant pour empecher sa reutilisation jusqu'a son expiration.

### 2.2 Perimetre

Services concernes :

- `web`
- `gateway`
- `user-service`

Si les microservices backend restent accessibles hors gateway en environnement local ou futur, la verification
d'invalidation devra etre partagee egalement par les services exposes.

### 2.3 Comportement attendu cote frontend

Le frontend doit proposer une action de deconnexion centralisee et non plus une simple suppression de token locale
depuis un composant isole.

Comportements obligatoires :

- Ajouter une action `logout` dans la couche applicative centrale d'authentification.
- Exposer la deconnexion depuis le header et depuis l'ecran profil.
- A l'appel de la deconnexion :
    - appeler le backend si un token est present ;
    - vider le token local ;
    - vider les donnees utilisateur en store ;
    - vider les erreurs d'authentification visibles ;
    - rediriger vers `/sign-in`.
- Si l'appel backend echoue pour une raison reseau, la deconnexion locale doit tout de meme se faire.
- Si le token est expire ou invalide, l'application doit basculer vers l'etat deconnecte sans boucle ni erreur visible
  parasite.

Comportements complementaires recommandes :

- centraliser la logique dans un `AuthFacadeService` ou equivalent ;
- ajouter un interceptor ou une gestion globale des `401` pour deconnecter l'utilisateur si le token n'est plus valide ;
- eviter toute duplication de logique de deconnexion dans les composants.

### 2.4 Comportement attendu cote backend

Le backend doit exposer un endpoint :

- `POST /auth/logout`

Regles :

- le token JWT est recupere depuis le header `Authorization: Bearer ...` ;
- si le token est absent ou invalide, la reponse doit etre `401` ;
- si le token est valide, le service l'invalide et retourne `204 No Content`.

### 2.5 Strategie technique recommandee

Comme l'authentification actuelle repose sur des JWT stateless, une vraie deconnexion backend impose une strategie
d'invalidation.

Strategie recommandee :

- introduire un stockage des tokens revoques ;
- cle recommandee : hash du token ;
- donnees minimales stockees :
    - `tokenHash`
    - `userId` ou `mail`
    - `revokedAt`
    - `expiresAt`
    - `reason`
- duree de conservation : jusqu'a l'expiration naturelle du JWT.

Stockage recommande :

- Redis en priorite si ajout infra acceptable ;
- a defaut, table SQL dediee dans `user-service`.

### 2.6 Verification de revocation

Apres ajout de la revocation, toute requete authentifiee doit verifier :

- signature JWT valide ;
- expiration JWT ;
- absence du token dans le registre des tokens revoques.

Implementation attendue :

- mutualiser cette verification dans un composant de securite partage ;
- faire porter ce controle au minimum par le `gateway` ;
- si des appels directs aux microservices sont possibles, appliquer aussi la verification dans les filtres JWT des
  services concernes.

### 2.7 Cas fonctionnels a couvrir

- un utilisateur connecte se deconnecte volontairement ;
- un utilisateur avec token expire est deconnecte automatiquement ;
- un utilisateur deconnecte ne peut plus appeler `/users/me` avec son ancien token ;
- un refresh du navigateur apres deconnexion ne doit pas rehydrater l'etat authentifie ;
- une double deconnexion consecutive ne doit pas provoquer d'erreur fonctionnelle.

### 2.8 Criteres d'acceptation

- le token n'est plus present en stockage local apres deconnexion ;
- le store utilisateur est vide apres deconnexion ;
- l'utilisateur est redirige vers `/sign-in` ;
- toute requete ulterieure avec l'ancien token renvoie `401` avec un code fonctionnel explicite ;
- la deconnexion fonctionne meme si l'API logout est indisponible.

## 3. Specs de validation des formulaires frontend et des DTO backend

### 3.1 Regles transverses

Toutes les entrees utilisateur doivent etre validees deux fois :

- une premiere fois cote frontend pour l'ergonomie ;
- une seconde fois cote backend pour la securite et l'integrite.

Regles globales :

- trim automatique des champs texte saisis ;
- interdiction des champs vides composes uniquement d'espaces ;
- messages d'erreur affiches par champ et non uniquement au niveau global ;
- blocage de la soumission tant que le formulaire est invalide ;
- en backend, usage systematique de `jakarta.validation` ;
- en backend, ajout de `@Valid` sur tous les `@RequestBody` et DTO imbriques.

### 3.2 Formulaire d'inscription

Frontend concerne :

- `sign-up.component.ts`

Etat actuel :

- tous les champs sont `required` ;
- `mail` n'utilise pas `Validators.email` ;
- aucun controle de format sur `phone`, `postalCode`, `password` ;
- pas de champ de confirmation de mot de passe.

Spec frontend :

- `firstname`
    - requis
    - longueur 2 a 50
    - lettres, espaces, apostrophes et tirets uniquement
- `lastname`
    - requis
    - longueur 2 a 50
    - memes regles que `firstname`
- `mail`
    - requis
    - format email valide
    - longueur max 254
- `password`
    - requis
    - longueur min 8
    - au moins 1 majuscule, 1 minuscule, 1 chiffre
- `confirmPassword`
    - nouveau champ frontend
    - requis
    - doit correspondre a `password`
- `phone`
    - requis
    - format telephone francais ou international restreint
- `addressLine`
    - requis
    - longueur 5 a 120
- `city`
    - requis
    - longueur 2 a 80
- `postalCode`
    - requis
    - format FR `^[0-9]{5}$`

Spec backend sur `SubscriptionDto` et `AddressDto` :

- `SubscriptionDto`
    - `firstname`: `@NotBlank`, `@Size(min = 2, max = 50)`, `@Pattern(...)`
    - `lastname`: `@NotBlank`, `@Size(min = 2, max = 50)`, `@Pattern(...)`
    - `mail`: `@NotBlank`, `@Email`, `@Size(max = 254)`
    - `password`: `@NotBlank`, `@Size(min = 8, max = 100)`, `@Pattern(...)`
    - `phone`: `@NotBlank`, `@Pattern(...)`
    - `address`: `@NotNull`, `@Valid`
- `AddressDto`
    - `adressLine1`: `@NotBlank`, `@Size(min = 5, max = 120)`
    - `city`: `@NotBlank`, `@Size(min = 2, max = 80)`
    - `postalCode`: `@NotBlank`, `@Pattern(regexp = "^[0-9]{5}$")`
    - `country`: `@NotBlank`, `@Size(min = 2, max = 2)` ou valeur par defaut `FR`

Comportement metier attendu :

- si le mail existe deja, retour d'une exception metier dediee ;
- si le DTO est invalide, retour d'une erreur de validation standardisee avec details par champ.

### 3.3 Formulaire de connexion

Frontend concerne :

- `sign-in.component.ts`

Etat actuel :

- `mail` a deja `required + email` ;
- `password` est seulement `required`.

Spec frontend :

- `mail`
    - requis
    - format email
- `password`
    - requis
    - longueur min 8

Spec backend sur `LoginDto` :

- `mail`: `@NotBlank`, `@Email`, `@Size(max = 254)`
- `password`: `@NotBlank`, `@Size(min = 8, max = 100)`

Comportement metier attendu :

- ne pas reveler si l'erreur vient du mail ou du mot de passe dans le message utilisateur final ;
- utiliser un code metier unique de type `INVALID_CREDENTIALS`.

### 3.4 Formulaire mot de passe oublie

Frontend concerne :

- `forgot-password.component.ts`

Etat actuel :

- validation correcte minimale sur le mail ;
- pas de normalisation commune des erreurs.

Spec frontend :

- `mail`
    - requis
    - format email
    - trim avant envoi

Spec backend sur `ResetPasswordMailDto` :

- `mail`: `@NotBlank`, `@Email`, `@Size(max = 254)`

Comportement metier attendu :

- deux options possibles :
    - mode securite fort : toujours renvoyer `202 Accepted` sans indiquer si le compte existe ;
    - mode explicite : conserver l'erreur `404` actuelle.

Recommendation :

- adopter le mode securite fort pour eviter l'enumeration des comptes.

### 3.5 Formulaire de reinitialisation du mot de passe

Frontend concerne :

- `reset-password.component.ts`

Etat actuel :

- controle de correspondance entre mots de passe deja present ;
- mot de passe uniquement `required`.

Spec frontend :

- `password`
    - requis
    - longueur min 8
    - au moins 1 majuscule, 1 minuscule, 1 chiffre
- `validatePassword`
    - requis
    - identique a `password`
- le token d'URL doit etre verifie comme present avant soumission

Spec backend sur `ResetPasswordConfirmDto` :

- `password`: `@NotBlank`, `@Size(min = 8, max = 100)`, `@Pattern(...)`
- `token`: `@NotBlank`

Comportement metier attendu :

- si le token est absent, invalide ou expire, retour d'une exception metier dediee ;
- si le mot de passe est identique a l'ancien et que la politique l'interdit, retour d'une exception metier dediee.

### 3.6 Formulaire de creation/modification de formule

Frontend concerne :

- `panel.component.ts`

Etat actuel :

- seuls `title` et `price` sont requis ;
- `description` est geree comme tableau cote front puis comme `String` cote API ;
- pas de controle sur prix, prix promotionnel ou coherence metier.

Spec frontend :

- `title`
    - requis
    - longueur 3 a 120
- `description`
    - requis
    - au moins un element non vide
- `price`
    - requis
    - nombre positif strict
- `promotionnalPrice`
    - optionnel
    - nombre positif ou nul
    - doit etre inferieur strictement a `price` si renseigne
- `code`
    - requis
    - boolean

Spec backend :

Recommendation structurelle :

- ne plus reutiliser le DTO de sortie `FormulaDto` comme DTO d'entree ;
- creer :
    - `CreateFormulaDto`
    - `UpdateFormulaDto`
    - `FormulaDto` pour la reponse

Validations minimales :

- `title`: `@NotBlank`, `@Size(min = 3, max = 120)`
- `description`: `@NotBlank`
- `price`: `@NotNull`, `@DecimalMin(value = "0.01")`
- `promotionnalPrice`: `@DecimalMin(value = "0.00")`
- validation metier supplementaire : `promotionnalPrice < price` si renseigne

Comportement metier attendu :

- si la formule n'existe pas, lever `FormulaNotFoundException` ;
- si les montants sont incoherents, lever `InvalidFormulaPriceException`.

### 3.7 Parametres et DTO secondaires a valider

Meme hors formulaires visuels, plusieurs entrees doivent etre durcies :

- `POST /payment/initier`
    - ne plus accepter `formulaId` brut sans validation ;
    - preferer un DTO de requete ou un `@RequestParam` valide en `Long` positif ;
- `PUT /users/formula`
    - valider `formulaId` en positif et obligatoire ;
- `POST /mails/forgot-password`
    - valider le DTO `ForgotPassword` dans le service de notification ;
- tous les DTO transmis via Kafka doivent aussi respecter des contraintes minimales de nullite et de format.

### 3.8 Gestion frontend des erreurs de validation

Le frontend doit afficher :

- les erreurs de validation champ par champ lorsque le backend les renvoie ;
- un message global uniquement pour les erreurs metier ou techniques ;
- un mapping centralise du contrat d'erreur API vers un modele frontend unique.

## 4. Specs de gestion d'exception commune

### 4.1 Objectif

Mettre en place une gestion des erreurs uniforme sur tous les microservices avec :

- une classe racine `ApiException` ;
- un contrat de reponse unique ;
- des exceptions metier explicites par domaine ;
- une integration propre cote frontend.

### 4.2 Probleme actuel

Etat constate :

- `user-service` dispose d'exceptions custom mais elles portent chacune `errorCode` et `HttpStatus` ;
- `plateforme-formula-service` capte toute exception et renvoie `400`, ce qui masque les vrais cas fonctionnels ;
- `plateforme-payment-service` lance des `RuntimeException` sans handler global ;
- `plateforme-notification-service` n'a pas de contrat d'erreur commun ;
- le frontend ne connait pas les `errorCode`.

### 4.3 Contrat de reponse unique

Le format de reponse d'erreur commun doit etre :

```json
{
  "timestamp": "2026-03-28T10:15:30Z",
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_ERROR",
  "message": "La requete est invalide",
  "path": "/auth/signup",
  "correlationId": "9d5d5f0a-6f5e-4c2b-8d77-1f4b12ab3321",
  "details": [
    {
      "field": "mail",
      "message": "must be a well-formed email address",
      "code": "EMAIL_INVALID"
    }
  ]
}
```

Champs obligatoires :

- `timestamp`
- `status`
- `error`
- `code`
- `message`
- `path`

Champs recommandes :

- `correlationId`
- `details`

### 4.4 Classe racine `ApiException`

La classe `ApiException` doit :

- etendre `RuntimeException` ;
- porter au minimum :
    - `String code`
    - `HttpStatus status`
    - `String message`
- permettre un constructeur avec cause technique optionnelle.

Exemple de structure cible :

- `ApiException`
- `ResourceNotFoundException`
- `ConflictException`
- `UnauthorizedException`
- `ForbiddenException`
- `ValidationException`
- exceptions metier specialisees par domaine

### 4.5 Partage entre microservices

Recommendation :

- creer un module Maven partage, par exemple `plateforme-common` ou `plateforme-shared-exception`.

Ce module doit contenir :

- `ApiException`
- `ApiErrorResponse`
- `ApiErrorDetail`
- un enum optionnel `ApiErrorCode`
- un handler commun pour services Spring MVC

Attention :

- le `gateway` est en WebFlux, donc le contrat doit etre le meme mais le handler/reactive adapter pourra etre specifique
  si necessaire.

### 4.6 Handler global attendu

Chaque service doit exposer un handler global qui :

- convertit toute `ApiException` vers le contrat commun ;
- convertit les erreurs de validation `MethodArgumentNotValidException` vers `VALIDATION_ERROR` ;
- convertit les erreurs techniques non prevues vers `INTERNAL_SERVER_ERROR` avec journalisation serveur ;
- ne retourne jamais de stacktrace au client ;
- loggue avec niveau adapte :
    - `warn` pour metier/validation
    - `error` pour technique

### 4.7 Exceptions metier minimales par service

`user-service`

- `UserAlreadyExistsException`
- `InvalidCredentialsException`
- `UserNotFoundException`
- `InvalidResetPasswordTokenException`
- `ExpiredResetPasswordTokenException`
- `RevokedTokenException`

`plateforme-formula-service`

- `FormulaNotFoundException`
- `InvalidFormulaPriceException`
- `FormulaAlreadyExistsException` si unicite future

`plateforme-payment-service`

- `PaymentNotFoundException`
- `PaymentInitializationException`
- `InvalidPaymentSignatureException`
- `PaymentProviderException`

`plateforme-notification-service`

- `MailDispatchException`
- `TemplateRenderingException`
- `InvalidNotificationPayloadException`

### 4.8 Codes de statut a normaliser

- `400` pour donnees invalides
- `401` pour non authentifie ou token invalide/revoque
- `403` pour acces refuse
- `404` pour ressource absente
- `409` pour conflit metier
- `422` possible si vous souhaitez distinguer validation syntaxique et regles metier
- `500` pour erreur technique non prevue

### 4.9 Integration frontend

Le frontend doit introduire un modele unique, par exemple `ApiExceptionModel`, avec :

- `status`
- `code`
- `message`
- `details`

Regles frontend :

- un service ou interceptor traduit toute `HttpErrorResponse` vers ce modele ;
- les effects NGRX ne doivent plus acceder directement a `err.error.message` sans normalisation ;
- les composants affichent un message global et, si disponible, les erreurs par champ.

### 4.10 Criteres d'acceptation

- tous les microservices renvoient le meme format d'erreur ;
- un DTO invalide renvoie une erreur de validation avec details par champ ;
- une exception metier renvoie un code stable et exploitable par le frontend ;
- une erreur technique inattendue renvoie `500` avec contrat standard ;
- le frontend affiche correctement le message ou les details sans logique specifique a chaque page.

## 5. Liste des points a rectifier ou solidifier observes

Sans detailler ici de specs completes, les sujets suivants meritent d'etre traites :

- la plateforme ne gere pas encore de refresh token ni de rotation de session ;
- l'uniformisation du contrat d'erreur entre appels HTTP et evenements Kafka n'est pas encore definie.

## 6. Decoupage recommande en lots

Pour execution, le chantier peut etre decoupe ainsi :

1. Standardiser le contrat d'erreur backend et frontend.
2. Ajouter les validations Bean Validation sur tous les DTO et `@Valid` dans les controllers.
3. Durcir les formulaires Angular et les messages par champ.
4. Ajouter la deconnexion centralisee frontend.
5. Ajouter les tests automatiques associes.
