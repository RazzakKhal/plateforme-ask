# plateforme-auto-ecole

Documentation backend du projet. Ce README couvre uniquement les microservices Java/Spring Boot et ignore volontairement le dossier `web`.

## Vue d'ensemble

Le dépôt contient 7 microservices backend orchestrés par le `pom.xml` racine :

| Microservice | Port | Rôle principal |
| --- | --- | --- |
| `plateforme-config-server` | `8071` | Centralise la configuration Spring Cloud Config depuis un dépôt Git externe. |
| `plateforme-eureka-server` | `8070` | Registre de découverte de services Eureka. |
| `gateway` | `8072` | Point d'entrée HTTP unique, routage vers les services backend. |
| `user-service` | `3001` | Authentification, gestion du compte utilisateur, reset de mot de passe, association formule/utilisateur. |
| `plateforme-formula-service` | `3002` | Gestion du catalogue des formules d'auto-école. |
| `plateforme-payment-service` | `3003` | Initialisation des paiements Monetico et traitement des retours de paiement. |
| `plateforme-notification-service` | `3004` | Envoi des mails de notification, en particulier pour le reset de mot de passe. |

## Architecture commune

Chaque microservice suit globalement la structure Spring Boot suivante :

```text
<service>/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/...         # code applicatif
    │   └── resources/       # configuration Spring, SQL, templates...
    └── test/java/...        # tests
```

Les services métier exposent aussi la famille d'endpoints Actuator via `/actuator/**`, car `management.endpoints.web.exposure.include: "*"` est activé dans leurs fichiers de configuration.

## Flux backend importants

### Authentification et reset de mot de passe

1. `user-service` expose les endpoints `/auth/**`.
2. `POST /auth/forgot-password` publie un message Kafka sur `password-reset-requested`.
3. `plateforme-notification-service` consomme ce message et envoie le mail de réinitialisation.

### Paiement et activation d'une formule

1. `plateforme-payment-service` crée un formulaire de paiement Monetico à partir de l'utilisateur courant et de la formule choisie.
2. Monetico rappelle `POST /payment/retour`.
3. Si le paiement est validé, `plateforme-payment-service` publie un message Kafka sur `send-communication`.
4. `user-service` consomme ce message et met à jour le `formulaId` de l'utilisateur concerné.

## Gateway

### Rôle

Le `gateway` est le point d'entrée HTTP du backend. Il route les requêtes vers les services enregistrés dans Eureka et réécrit les chemins avant transfert.

### Architecture de dossiers

```text
gateway/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/gateway/
    │   │   ├── GatewayApplication.java
    │   │   ├── feign/
    │   │   ├── security/
    │   │   └── services/
    │   └── resources/
    │       └── application.yml
    └── test/java/
```

### Endpoints exposés

| Méthode | Endpoint | Description |
| --- | --- | --- |
| `ANY` | `/user-service/**` | Route toutes les requêtes vers `USER-SERVICE` après réécriture du préfixe. Exemple : `/user-service/auth/signin` devient `/auth/signin` côté service. |
| `ANY` | `/formula-service/**` | Route toutes les requêtes vers `FORMULA-SERVICE`. |
| `ANY` | `/payment-service/**` | Route toutes les requêtes vers `PAYMENT-SERVICE`. |
| `ANY` | `/notification-service/**` | Route toutes les requêtes vers `NOTIFICATION-SERVICE`. |
| `GET` | `/actuator/**` | Endpoints techniques du gateway. `endpoint.gateway.access: unrestricted` active aussi les endpoints Actuator propres à Spring Cloud Gateway. |

## user-service

### Rôle

`user-service` gère l'authentification JWT, la création de compte, la consultation de l'utilisateur connecté, la validation de token, le reset de mot de passe et la liaison d'une formule à un utilisateur.

### Architecture de dossiers

```text
user-service/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/user_service/
    │   │   ├── auditing/
    │   │   ├── controllers/
    │   │   ├── dtos/
    │   │   ├── enums/
    │   │   ├── exceptions/
    │   │   ├── functions/
    │   │   ├── handlers/
    │   │   ├── interfaces/
    │   │   ├── mappers/
    │   │   ├── models/
    │   │   ├── repositories/
    │   │   ├── security/
    │   │   └── services/
    │   └── resources/
    │       ├── application.yaml
    │       ├── static/
    │       └── templates/
    └── test/java/
```

### Endpoints exposés

Base locale : `http://localhost:3001`  
Via gateway : `http://localhost:8072/user-service`

| Méthode | Endpoint | Accès | Description |
| --- | --- | --- | --- |
| `POST` | `/auth/signin` | Public | Authentifie un utilisateur à partir de `mail` et `password`, puis renvoie un `TokenDto` contenant l'email, les rôles et le JWT. |
| `POST` | `/auth/signup` | Public | Crée un compte utilisateur à partir de `firstname`, `lastname`, `mail`, `password`, `phone` et `address`, chiffre le mot de passe puis renvoie un JWT. |
| `POST` | `/auth/forgot-password` | Public | Recherche l'utilisateur par email et publie un message Kafka pour déclencher l'envoi du lien de réinitialisation. Réponse HTTP `202 Accepted`. |
| `POST` | `/auth/reset-password` | Public | Vérifie le token de reset, remplace le mot de passe par la nouvelle valeur chiffrée puis renvoie un nouveau JWT. |
| `GET` | `/users/me` | Authentifié | Renvoie le profil de l'utilisateur courant sous forme de `UserDto`. |
| `GET` | `/users/validate` | Public | Valide le JWT reçu dans le header `Authorization` et renvoie un objet `{ mail, roles }` si le token est correct. |
| `PUT` | `/users/formula?formulaId={id}` | Authentifié | Associe la formule indiquée à l'utilisateur actuellement authentifié. Réponse HTTP `204 No Content`. |
| `GET` | `/users/myenv` | Authentifié | Retourne la valeur de la propriété Spring `test`, utile pour vérifier la configuration externe chargée depuis le config server. |
| `GET` | `/actuator/**` | Public | Endpoints techniques Spring Boot Actuator. |
| `GET` | `/v3/api-docs` | Authentifié | Spécification OpenAPI générée automatiquement. Ces routes ne sont pas explicitement ouvertes dans `SecurityConfig`. |
| `GET` | `/swagger-ui/index.html` | Authentifié | Interface Swagger UI du service. Ces routes ne sont pas explicitement ouvertes dans `SecurityConfig`. |

### Remarques techniques

- Le service consomme le topic Kafka `send-communication` via la fonction `saveFormula` pour mettre à jour `formulaId` après un paiement réussi.
- Le service publie sur `password-reset-requested` via le binding `send-forgot-password-link-out-0`.

## plateforme-formula-service

### Rôle

`plateforme-formula-service` gère le catalogue des formules vendues par la plateforme : création, modification, suppression et consultation.

### Architecture de dossiers

```text
plateforme-formula-service/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/formula_service/
    │   │   ├── controllers/
    │   │   ├── dtos/
    │   │   ├── exceptions/
    │   │   ├── mappers/
    │   │   ├── models/
    │   │   ├── repositories/
    │   │   ├── security/
    │   │   ├── services/
    │   │   └── shared/
    │   └── resources/
    │       ├── application.yaml
    │       └── data.sql
    └── test/java/
```

### Endpoints exposés

Base locale : `http://localhost:3002`  
Via gateway : `http://localhost:8072/formula-service`

| Méthode | Endpoint | Accès | Description |
| --- | --- | --- | --- |
| `POST` | `/formulas` | Authentifié + rôle `ADMIN` | Crée une formule à partir d'un `FormulaDto` (`title`, `description`, `price`, `code`, `promotionnalPrice`) et renvoie la formule créée. |
| `PUT` | `/formulas/{id}` | Authentifié + rôle `ADMIN` | Met à jour la formule d'identifiant `{id}` puis renvoie la version mise à jour. |
| `DELETE` | `/formulas/{id}` | Authentifié + rôle `ADMIN` | Supprime la formule ciblée. Réponse HTTP `204 No Content`. |
| `GET` | `/formulas/{id}` | Authentifié | Renvoie le détail d'une formule précise. |
| `GET` | `/formulas` | Public | Renvoie la liste complète des formules. C'est la seule route métier explicitement publique dans la configuration de sécurité. |
| `GET` | `/formulas/authentication` | Authentifié | Endpoint de diagnostic qui renvoie l'objet `Authentication` résolu par Spring Security. |
| `GET` | `/actuator/**` | Public | Endpoints techniques Spring Boot Actuator. |
| `GET` | `/v3/api-docs` | Authentifié | Spécification OpenAPI générée automatiquement. Ces routes ne sont pas explicitement ouvertes dans `SecurityConfig`. |
| `GET` | `/swagger-ui/index.html` | Authentifié | Interface Swagger UI du service. Ces routes ne sont pas explicitement ouvertes dans `SecurityConfig`. |

## plateforme-payment-service

### Rôle

`plateforme-payment-service` pilote l'intégration Monetico : génération du formulaire de paiement, persistance d'une trace locale du paiement, validation du retour Monetico et publication d'un événement de paiement réussi.

### Architecture de dossiers

```text
plateforme-payment-service/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/payment_service/
    │   │   ├── configuration/
    │   │   ├── controllers/
    │   │   ├── dtos/
    │   │   ├── enums/
    │   │   ├── feign/
    │   │   ├── infrastructure/
    │   │   ├── mappers/
    │   │   ├── models/
    │   │   ├── repositories/
    │   │   ├── responses/
    │   │   ├── security/
    │   │   └── services/
    │   └── resources/
    │       └── application.yaml
    └── test/java/
```

### Endpoints exposés

Base locale : `http://localhost:3003`  
Via gateway : `http://localhost:8072/payment-service`

| Méthode | Endpoint | Accès | Description |
| --- | --- | --- | --- |
| `GET` | `/` | Authentifié | Endpoint de test qui retourne la chaîne `test reussi`. Il n'est pas whiteliste dans `SecurityConfig`. |
| `POST` | `/payment/initier?formulaId={id}` | Authentifié | Récupère l'utilisateur courant, charge la formule demandée via Feign, calcule le prix, génère le formulaire Monetico (`PaymentFormDto`), crée un enregistrement de paiement local puis renvoie les champs à poster vers Monetico. |
| `POST` | `/payment/retour` | Public | Endpoint de callback Monetico. Vérifie la MAC/HMAC reçue, marque le paiement en `SUCCESS`, `FAILED` ou `INVALID_SIGNATURE`, puis publie un événement Kafka si le paiement est réussi. La réponse renvoyée à Monetico est au format attendu `version=2` / `cdr=0|1`. |
| `GET` | `/actuator/**` | Public | Endpoints techniques Spring Boot Actuator. |

### Remarques techniques

- En cas de succès, le service publie un `PaymentDto` sur le binding Kafka `sendCommunication-out-0`.
- Le service interroge `user-service` via `GET /users/me` et `plateforme-formula-service` via `GET /formulas/{id}`.

## plateforme-notification-service

### Rôle

`plateforme-notification-service` envoie les mails liés à la plateforme. Dans l'état actuel du code, il est centré sur le mail de réinitialisation de mot de passe.

### Architecture de dossiers

```text
plateforme-notification-service/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/notification_service/
    │   │   ├── configurations/
    │   │   ├── controllers/
    │   │   ├── dtos/
    │   │   ├── records/
    │   │   └── services/
    │   └── resources/
    │       └── application.yaml
    └── test/java/
```

### Endpoints exposés

Base locale : `http://localhost:3004`  
Via gateway : `http://localhost:8072/notification-service`

| Méthode | Endpoint | Accès | Description |
| --- | --- | --- | --- |
| `POST` | `/mails/forgot-password` | Public | Déclenche l'envoi d'un mail de reset à partir d'un body `ForgotPassword { mail, token }`. Réponse HTTP `202 Accepted`. |
| `GET` | `/actuator/**` | Public | Endpoints techniques Spring Boot Actuator. |

### Remarques techniques

- Le service consomme aussi le topic Kafka `password-reset-requested` via la fonction `sendResetLink`.
- Le contrôleur HTTP et le consumer Kafka réutilisent le même `MailService`.

## plateforme-config-server

### Rôle

`plateforme-config-server` est un serveur Spring Cloud Config. Il lit la configuration centralisée depuis le dépôt Git `git@github.com:RazzakKhal/plateforme-env.git` et la sert aux autres microservices.

### Architecture de dossiers

```text
plateforme-config-server/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/configServer/
    │   │   └── ConfigServerApplication.java
    │   └── resources/
    │       └── application.yml
    └── test/java/
```

### Endpoints exposés

Base locale : `http://localhost:8071`

| Méthode | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/{application}/{profile}` | Renvoie la configuration agrégée d'une application pour un profil donné. Exemple typique : `/user-service/prod`. |
| `GET` | `/{application}/{profile}/{label}` | Même comportement, mais en ciblant explicitement une branche ou un label Git. |
| `GET` | `/{application}-{profile}.yml` | Renvoie la configuration sous forme YAML. |
| `GET` | `/{application}-{profile}.properties` | Renvoie la configuration au format `.properties`. |
| `GET` | `/actuator/**` | Endpoints techniques Spring Boot Actuator du config server. |

## plateforme-eureka-server

### Rôle

`plateforme-eureka-server` est le registre de services. Les autres microservices s'y enregistrent et le gateway l'utilise pour résoudre `USER-SERVICE`, `FORMULA-SERVICE`, `PAYMENT-SERVICE` et `NOTIFICATION-SERVICE`.

### Architecture de dossiers

```text
plateforme-eureka-server/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/bookNDrive/eureka_service/
    │   │   └── EurekaServiceApplication.java
    │   └── resources/
    │       └── application.yml
    └── test/java/
```

### Endpoints exposés

Base locale : `http://localhost:8070`

| Méthode | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/` | Affiche l'interface web Eureka Server avec la liste des instances enregistrées. |
| `GET` | `/eureka/**` | Endpoints techniques de consultation du registre Eureka. |
| `POST` | `/eureka/**` | Endpoints techniques utilisés par les clients Eureka pour l'enregistrement et le heartbeat. |
| `PUT` | `/eureka/**` | Endpoints techniques utilisés par les clients Eureka pour la gestion du registre. |
| `DELETE` | `/eureka/**` | Endpoints techniques utilisés pour la désinscription d'instances. |
| `GET` | `/actuator/**` | Endpoints techniques Spring Boot Actuator du serveur Eureka. |

## Résumé des endpoints métier

### Authentification et utilisateur

- `POST /auth/signin`
- `POST /auth/signup`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `GET /users/me`
- `GET /users/validate`
- `PUT /users/formula`
- `GET /users/myenv`

### Formules

- `POST /formulas`
- `PUT /formulas/{id}`
- `DELETE /formulas/{id}`
- `GET /formulas/{id}`
- `GET /formulas`
- `GET /formulas/authentication`

### Paiement

- `GET /`
- `POST /payment/initier`
- `POST /payment/retour`

### Notification

- `POST /mails/forgot-password`
