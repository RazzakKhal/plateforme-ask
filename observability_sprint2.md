# Observability Sprint 2

## Objectif du sprint

Le sprint 2 doit livrer une brique de logs centralises exploitable dans Grafana, sans attendre encore le tracing distribue du sprint 3.

Le resultat attendu est simple :

- les microservices backend ecrivent des logs JSON structures sur la console
- `Loki` stocke les logs centralises
- `Grafana Alloy` collecte les logs Docker et les envoie vers Loki
- `Grafana` permet de rechercher les logs par service, niveau et periode

Ce sprint sert a repondre rapidement a trois questions :

- quelle erreur applicative est en train de se produire
- dans quel service elle apparait
- quels logs precedents et suivants permettent de comprendre le probleme

## Ce que l'on ajoute et a quoi cela sert

### 1. Logs JSON structures

But :

- produire des logs lisibles par une machine, pas seulement par un humain

Pourquoi c'est necessaire :

- Loki et Grafana Logs deviennent beaucoup plus utiles quand chaque champ est explicite
- un log texte libre est difficile a filtrer proprement
- un log JSON permet de retrouver facilement `service`, `level`, `logger`, `message` et `exception`

### 2. `Loki`

But :

- centraliser et stocker les logs de la plateforme

Pourquoi c'est necessaire :

- sans Loki, il faut ouvrir les logs container un par un
- Loki fournit un stockage adapte aux logs et un moteur de requetes integre a Grafana
- c'est la brique qui transforme des sorties console Docker en historique interrogeable

### 3. `Grafana Alloy`

But :

- collecter les logs des containers Docker et les pousser vers Loki

Pourquoi c'est necessaire :

- Grafana ne lit pas directement les logs Docker
- Loki n'aspire pas tout seul les containers
- Alloy joue le role d'agent de collecte moderne pour Grafana, a la place de Promtail

### 4. Datasource Loki dans Grafana

But :

- rendre les logs consultables directement depuis l'interface deja en place

Pourquoi c'est necessaire :

- le depot a deja Grafana pour les metriques
- ajouter Loki dans la meme interface permet de croiser metriques et logs sans changer d'outil

### 5. Requetes et vues de logs de base

But :

- fournir des recherches utiles des le premier jour

Pourquoi c'est necessaire :

- une stack de logs sans requetes de depart reste peu exploitee
- l'equipe doit pouvoir retrouver tout de suite les erreurs du `gateway`, les exceptions du `payment-service` et le bruit SQL excessif

## Perimetre du sprint 2

Logs applicatifs a centraliser :

- `gateway`
- `user-service`
- `formula-service`
- `payment-service`
- `notification-service`

Logs techniques a centraliser des ce sprint si le cout reste faible :

- `config-server`
- `eureka-server`

Logs d'infrastructure a envisager ensuite, mais pas obligatoires pour la premiere livraison :

- `rabbitmq`
- `kafka`
- `db`
- `db-formula`
- `db-payment`

Pourquoi ce perimetre :

- la valeur la plus immediate vient des logs des services qui portent les parcours utilisateurs
- les logs infra sont utiles, mais ils peuvent faire monter le volume tres vite
- commencer par les services applicatifs limite la complexite et facilite le tuning des labels

Le frontend Angular n'entre pas dans ce sprint.

## Etat actuel du depot

Le depot est deja bien prepare pour demarrer ce sprint :

- la phase metriques existe deja dans `docker-compose/dev`, `docker-compose/local` et `docker-compose/prod`
- `Prometheus` et `Grafana` sont deja deploies dans ces environnements
- les services backend exposent deja les endpoints Actuator utiles pour le sprint 1

En revanche, il manque encore les briques specifiques aux logs centralises :

- aucun `Loki` n'est present dans les environnements Compose
- aucun `Grafana Alloy` n'est present pour collecter les logs Docker
- les services backend n'ont pas encore de configuration standard de logs JSON
- le depot ne contient pas encore de dependances ou de configuration de tracing distribue

Point important a prendre en compte :

- `user-service`, `plateforme-formula-service` et `plateforme-payment-service` ont encore `spring.jpa.show-sql: true` dans leur configuration principale

Conclusion :

- la collecte des metriques est deja en place
- le sprint 2 doit se brancher sur cette base existante
- il faut d'abord standardiser la sortie des applications, puis ajouter `Loki + Alloy`, puis exposer les logs dans Grafana

## Choix d'implementation recommandes

### 1. Collecter les logs depuis `stdout/stderr`

Je recommande de ne pas creer de fichiers de logs dans les containers.

Pourquoi ce choix est le bon pour ce depot :

- toute la plateforme est lancee en Docker Compose
- Docker sait deja capturer `stdout` et `stderr`
- Alloy sait lire directement les logs des containers via Docker
- on evite les volumes supplementaires, la rotation locale et les chemins de fichiers differents selon les services

Effet attendu :

- chaque service continue a logger sur la console
- mais cette console devient une source centralisee et structuree

### 2. Utiliser le structured logging natif de Spring Boot 3.4.1

Je recommande de partir sur la fonctionnalite native de Spring Boot plutot que d'ajouter un encodeur tiers.

Pourquoi ce choix est le bon :

- `ask-bom` porte deja Spring Boot `3.4.1`
- Spring Boot 3.4 supporte nativement des formats JSON structures
- cela evite d'ajouter une dependance supplementaire juste pour le logging JSON
- la maintenance est plus simple et reste dans les conventions Spring Boot

Format recommande :

- `ecs`

Pourquoi `ecs` :

- le format contient deja un bloc `service`
- il reste tres lisible dans Loki
- il est adaptee a une plateforme composee de plusieurs microservices

## Plan d'implementation

## Etape 1 - Standardiser les logs JSON sur tous les backends

Fichiers a modifier :

- `gateway/src/main/resources/application.yml`
- `user-service/src/main/resources/application.yaml`
- `plateforme-formula-service/src/main/resources/application.yaml`
- `plateforme-payment-service/src/main/resources/application.yaml`
- `plateforme-notification-service/src/main/resources/application.yaml`
- `plateforme-config-server/src/main/resources/application.yml`
- `plateforme-eureka-server/src/main/resources/application.yml`

Configuration cible recommandee :

```yaml
logging:
  structured:
    format:
      console: ecs
    ecs:
      service:
        name: ${spring.application.name}
        environment: ${SPRING_PROFILES_ACTIVE:local}
  level:
    root: INFO
```

Pourquoi cette etape existe :

- tant que les applications emettent des logs texte heterogenes, Alloy et Loki n'apportent qu'une valeur partielle
- il faut que tous les services produisent la meme structure de base
- `service.name` et `level` doivent etre disponibles de facon uniforme

Effet attendu :

- chaque ligne de log devient un objet JSON unique
- les champs standards deviennent filtrables dans Loki

Point d'attention :

- il faut garder la sortie sur la console
- il ne faut pas basculer vers `logging.file.*` dans les containers

## Etape 2 - Preparer la correlation future sans attendre le tracing distribue

But de l'etape :

- faire en sorte que le sprint 2 n'empeche pas la correlation `traceId` / `spanId` du sprint 3

Pourquoi cette etape existe :

- le depot n'a pas encore Micrometer Tracing ni OTLP
- il serait faux de promettre des `traceId` remplis des ce sprint
- en revanche, le JSON structure de Spring Boot inclut automatiquement les paires presentes dans le MDC

Ce qu'il faut faire concretement :

- conserver un format JSON compatible MDC
- ne pas ecrire de pattern console personnalise qui ferait disparaitre les futures cles de correlation
- si vous ajoutez un identifiant technique temporaire avant le sprint 3, utilisez plutot un `requestId` que des faux `traceId`

Effet attendu :

- le sprint 3 pourra ajouter le tracing sans reouvrir tout le chantier logs
- les futurs champs `traceId` et `spanId` remonteront naturellement dans les logs une fois le tracing active

Point important :

- la correlation inter-services complete n'est pas un objectif du sprint 2
- elle sera livree au sprint 3

## Etape 3 - Reduire le bruit des logs avant centralisation

Fichiers a verifier en priorite :

- `user-service/src/main/resources/application.yaml`
- `plateforme-formula-service/src/main/resources/application.yaml`
- `plateforme-payment-service/src/main/resources/application.yaml`

Actions a faire :

- supprimer ou desactiver `spring.jpa.show-sql: true` dans les configurations principales
- reserver les logs SQL detailles au local ou au debug explicite
- garder un niveau `INFO` sobre par defaut

Pourquoi cette etape existe :

- centraliser des logs trop verbeux fait monter le volume, le bruit et le cout d'analyse
- les logs SQL continus masquent rapidement les erreurs applicatives utiles
- le `payment-service` ne doit pas noyer les evenements metier dans du bruit ORM

Effet attendu :

- Loki reste utile pour l'investigation
- les requetes d'erreur ne sont pas polluees par du SQL permanent

## Etape 4 - Ajouter `Loki` dans `docker-compose/dev`

Fichiers a creer ou modifier :

- `docker-compose/dev/docker-compose.yml`
- `docker-compose/dev/loki/loki-config.yml`

Role de `Loki` dans ce depot :

- recevoir les logs collectes par Alloy
- les indexer avec peu de labels
- les rendre interrogeables depuis Grafana

Pourquoi on commence par `dev` :

- le depot a deja une structure `dev`, `local` et `prod`
- il vaut mieux valider le pipeline de logs une premiere fois dans `dev`
- la duplication vers `local` et `prod` ne doit venir qu'apres verification

Configuration a prevoir :

- un service `loki`
- un volume de persistance
- un montage de `loki-config.yml`
- une exposition du port `3100`

Effet attendu :

- Alloy dispose d'une cible unique pour pousser les logs
- Grafana peut ensuite se connecter a Loki via son API HTTP

## Etape 5 - Ajouter `Grafana Alloy` pour collecter les logs Docker

Fichiers a creer ou modifier :

- `docker-compose/dev/docker-compose.yml`
- `docker-compose/dev/alloy/config.alloy`

Role d'Alloy :

- decouvrir les containers Docker
- lire leurs logs
- enrichir les entrees avec des labels utiles
- pousser le tout vers Loki

Pourquoi Alloy est necessaire ici :

- le repo tourne deja en containers
- la source la plus fiable est donc le moteur Docker lui-meme
- cela evite de modifier les images applicatives uniquement pour l'expedition des logs

Montages et acces a prevoir :

- acces au socket Docker
- fichier de configuration Alloy monte en lecture seule

Principes de configuration recommandes :

- discovery Docker pour lister les containers
- collecte des logs Docker pour les services applicatifs
- etiquettes stables basees sur les metadonnees Docker Compose

Labels utiles a extraire :

- nom du service Compose
- nom du container
- environnement
- stream `stdout` ou `stderr`

Pourquoi ces labels precis :

- ils ont une cardinalite faible
- ils suffisent pour les recherches principales
- ils evitent d'indexer des valeurs trop variables comme `userId`, `email`, `traceId` ou message d'erreur

Effet attendu :

- une requete Grafana du type `{service_name="payment-service"}` ou equivalent devient possible
- les logs sont consultables sans ouvrir `docker logs` service par service

## Etape 6 - Choisir soigneusement ce qui devient un label Loki

But de l'etape :

- eviter une explosion de cardinalite et un Loki difficile a exploiter

Pourquoi cette etape existe :

- dans Loki, tous les champs ne doivent pas devenir des labels
- plus il y a de labels variables, plus l'index grossit et plus les requetes deviennent couteuses

Je recommande de mettre en labels seulement :

- `service`
- `container`
- `environment`
- `stream`

Je recommande de laisser dans le JSON, sans les transformer en labels :

- `message`
- `exception`
- `stacktrace`
- `logger`
- `requestId`
- futurs `traceId` et `spanId`
- toutes les donnees metier

Effet attendu :

- les recherches restent performantes
- les champs riches restent disponibles dans le contenu du log

## Etape 7 - Provisionner Loki comme datasource Grafana

Fichier a creer :

- `docker-compose/dev/grafana/provisioning/datasources/loki.yml`

Role :

- declarer automatiquement Loki dans Grafana

Exemple de structure :

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

Pourquoi cette etape existe :

- le depot provisionne deja Prometheus automatiquement
- il faut conserver la meme logique pour Loki
- une datasource configuree a la main n'est pas reproductible

Effet attendu :

- Grafana affiche les logs sans configuration manuelle post-deploiement

## Etape 8 - Ajouter des requetes et vues de logs de depart

Livrables recommandes :

- une vue d'exploration des erreurs par service
- une vue ciblee sur le `gateway`
- une vue ciblee sur le `payment-service`

Pourquoi cette etape existe :

- le sprint 2 doit livrer de la valeur visible, pas seulement des containers supplementaires
- les premiers besoins du projet sont deja connus : erreurs API, erreurs de paiement, timeout et bruit SQL

Recherches utiles a preparer :

- logs `ERROR` par service
- exceptions par type sur `user-service` et `payment-service`
- erreurs `5xx` ou timeout du `gateway`
- logs de paiement contenant un statut ou un echec fonctionnel

Effet attendu :

- l'equipe sait immediatement ou regarder lors d'un incident
- Grafana devient le point d'entree unique pour l'analyse

## Etape 9 - Etendre ensuite a `local` puis `prod`

Fichiers a adapter apres validation en `dev` :

- `docker-compose/local/docker-compose.yml`
- `docker-compose/local/grafana/provisioning/datasources/loki.yml`
- `docker-compose/local/loki/loki-config.yml`
- `docker-compose/local/alloy/config.alloy`
- `docker-compose/prod/docker-compose.yml`
- `docker-compose/prod/grafana/provisioning/datasources/loki.yml`
- `docker-compose/prod/loki/loki-config.yml`
- `docker-compose/prod/alloy/config.alloy`

Pourquoi cette etape existe :

- le repo a deja une declinaison par environnement
- la centralisation des logs doit suivre la meme organisation
- cela evite un environnement `dev` en avance et des environnements `local` ou `prod` oublies

Point d'attention pour `prod` :

- definir une retention explicite
- proteger les acces Grafana
- revoir plus strictement les logs sensibles

## Hygiene et securite des logs

Ce sprint doit inclure une revue explicite de ce qui ne doit pas partir dans Loki.

Il ne faut pas logger :

- le header `Authorization`
- les JWT bruts
- les mots de passe
- les secrets techniques
- les payloads de paiement bruts
- les emails ou identifiants sensibles si ce n'est pas indispensable

Pourquoi cette revue est obligatoire :

- la centralisation rend les donnees plus faciles a retrouver
- un mauvais log local devient un mauvais log visible partout
- les services `gateway`, `user-service` et `payment-service` sont particulierement sensibles

Ce qu'il faut verifier dans le code :

- les messages de logs manuels dans les controllers, filters et services
- les exceptions qui pourraient exposer des payloads complets
- les logs de securite autour des JWT

## Ordre concret de realisation

1. Passer les services backend en JSON structure sur la console
2. Reduire le bruit SQL et confirmer les niveaux de logs par defaut
3. Ajouter `Loki` dans `docker-compose/dev`
4. Ajouter `Grafana Alloy` dans `docker-compose/dev`
5. Configurer les labels utiles et limiter la cardinalite
6. Provisionner la datasource Loki dans Grafana
7. Verifier les recherches de base dans Grafana
8. Dupliquer ensuite vers `local`
9. Dupliquer enfin vers `prod`

Pourquoi cet ordre :

- il commence par rendre les applications propres a collecter
- ensuite seulement il branche la chaine de collecte
- il evite d'alimenter Loki avec des logs mal structures ou trop verbeux

## Verification attendue a la fin du sprint

### Verification applicative

Pour chaque backend :

- les logs consoles sont en JSON sur une seule ligne
- `service.name` est present
- `level` et `message` sont presents
- les exceptions restent lisibles dans le JSON

### Verification Alloy

Dans Alloy :

- les containers applicatifs sont bien decouverts
- les logs sont bien lus depuis Docker
- l'envoi vers Loki ne remonte pas d'erreurs

### Verification Loki

Dans Loki ou Grafana Explore :

- les logs du `gateway` sont consultables
- les logs du `payment-service` sont consultables
- un filtre par service retourne bien uniquement les lignes attendues

### Verification Grafana

Dans Grafana :

- la datasource Loki est chargee automatiquement
- on peut executer des requetes de logs sans configuration manuelle
- les vues de depart permettent de retrouver une erreur en moins de quelques minutes

## Limites volontaires du sprint 2

Ce sprint ne doit pas encore traiter completement :

- le tracing distribue
- la correlation de bout en bout via `traceId` entre tous les services
- l'alerting sur logs
- la retention fine et l'optimisation avancee de Loki
- la collecte exhaustive des logs infra

Pourquoi on les differe :

- ce sont des sujets utiles, mais plus couteux
- il faut d'abord valider la qualite et le volume des logs applicatifs
- la correlation forte viendra naturellement avec le sprint 3

## Mon avis sur la meilleure implementation

La meilleure livraison sprint 2 pour ce depot est :

- activer le structured logging natif de Spring Boot 3.4.1 sur tous les backends
- garder les logs sur `stdout/stderr`
- ajouter `Loki` et `Grafana Alloy` d'abord dans `docker-compose/dev`
- limiter les labels Loki a quelques dimensions stables
- desactiver le bruit SQL permanent avant ingestion
- provisionner `Loki` automatiquement dans Grafana

Ce choix est volontairement pragmatique :

- il colle a la structure actuelle du repo
- il ne demande pas de refondre les images applicatives
- il prepare proprement le sprint 3 sans le melanger avec le tracing

## References utiles

- Spring Boot logging: https://docs.spring.io/spring-boot/reference/features/logging.html
- Grafana Alloy et logs Docker: https://grafana.com/docs/alloy/latest/monitor/monitor-docker-containers/
- Loki datasource Grafana: https://grafana.com/docs/grafana/latest/datasources/loki/
- Promtail deprecation / migration vers Alloy: https://grafana.com/docs/loki/latest/send-data/promtail/
