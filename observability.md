# Plan d'observabilite avec Grafana

## Contexte du depot

L'application est deja bien positionnee pour demarrer l'observabilite :

- les microservices Spring Boot exposent deja `Actuator`
- les endpoints de health sont utilises par `docker-compose/dev/docker-compose.yml`
- l'architecture est distribuee : `gateway`, `user-service`, `formula-service`, `payment-service`, `notification-service`, `config-server`, `eureka-server`
- les dependances techniques critiques existent deja : PostgreSQL, RabbitMQ, Kafka

En revanche, il manque encore les trois briques d'observabilite exploitables dans Grafana :

- des metriques scrapeables et consolidees
- des logs centralises et interrogeables
- des traces distribuees entre services

## Recommandation cible

Je te conseille une stack Grafana OSS simple, progressive et adaptee a ton projet :

- `Prometheus` pour collecter les metriques Spring Boot
- `Grafana` pour les dashboards et alertes
- `Loki` pour les logs applicatifs et infra
- `Grafana Alloy` comme agent de collecte
- `Tempo` pour les traces distribuees

Pourquoi ce choix :

- c'est coherent avec Spring Boot + Micrometer
- c'est simple a lancer en Docker Compose
- Grafana permet ensuite de croiser metriques, logs et traces dans une seule interface
- `Promtail` ne doit pas etre choisi pour un nouveau chantier : il est deprecie, donc il vaut mieux partir directement sur `Grafana Alloy`

## Strategie de mise en place

Je recommande un deploiement en 4 phases, pour livrer vite sans complexifier toute la plateforme d'un coup.

### Phase 1 - Metriques et dashboards de base

Objectif : avoir une visibilite immediate sur la sante des services et leurs temps de reponse.

Travaux :

- ajouter `micrometer-registry-prometheus` dans chaque microservice backend
- exposer proprement `/actuator/prometheus`
- durcir la configuration `management` pour ne plus laisser `include: "*"` en production
- ajouter un `docker-compose/observability` ou etendre `docker-compose/dev` avec :
  - `prometheus`
  - `grafana`
- configurer Prometheus pour scraper :
  - `gateway:8072`
  - `user-service:3001`
  - `formula-service:3002`
  - `payment-service:3003`
  - `notification-service:3004`
  - optionnellement `config-server:8071`
  - optionnellement `eureka-server:8070`
- provisionner Grafana avec une datasource Prometheus

Dashboards minimums a creer :

- disponibilite des services
- latence HTTP par service et par endpoint
- taux d'erreur HTTP 4xx/5xx
- consommation CPU, memoire JVM, threads, GC
- connexions HTTP entrantes sur le gateway

Valeur attendue :

- tu sais tout de suite quel service degrade
- tu peux distinguer un probleme reseau, JVM ou applicatif
- tu obtiens une base saine avant d'ajouter logs et traces

### Phase 2 - Logs centralises

Objectif : pouvoir rechercher les erreurs sans ouvrir les logs container un par un.

Travaux :

- standardiser les logs Spring au format JSON
- injecter dans les logs les champs suivants :
  - `service.name`
  - `traceId`
  - `spanId`
  - `level`
  - `logger`
  - `exception`
- deployer `Loki`
- deployer `Grafana Alloy` pour collecter :
  - logs Docker des microservices
  - logs de RabbitMQ, Kafka, PostgreSQL si utile
- provisionner Grafana avec une datasource Loki

Bonnes pratiques importantes :

- ne pas logger les tokens JWT
- ne pas logger les payloads de paiement bruts
- masquer emails, secrets, cles et identifiants sensibles
- limiter le bruit des logs SQL en dehors du debug local

Dashboards et requetes utiles :

- erreurs applicatives par service
- exceptions par type
- logs du `payment-service` filtres par statut de paiement
- logs du `gateway` pour les erreurs 5xx et timeouts

### Phase 3 - Traces distribuees

Objectif : suivre une requete du `gateway` jusqu'aux services internes.

Travaux :

- activer le tracing Spring Boot avec Micrometer Tracing
- exporter les spans en OTLP vers Alloy ou directement vers Tempo
- deployer `Tempo`
- propager les contextes de trace entre :
  - `gateway`
  - appels HTTP inter-services
  - Kafka
  - RabbitMQ si utilise dans les parcours critiques
- enrichir les spans sur les cas metier importants :
  - login
  - inscription
  - achat de formule
  - creation de paiement
  - emission d'evenement outbox
  - envoi d'email

Cas d'usage tres concrets pour ton projet :

- comprendre si une lenteur vient du `gateway`, du `user-service` ou du `payment-service`
- suivre un echec du parcours de paiement de bout en bout
- correler une erreur email avec l'evenement source dans Kafka

### Phase 4 - Alerting et SLO

Objectif : rendre l'observabilite actionnable.

Travaux :

- definir 5 a 8 alertes prioritaires
- brancher Grafana Alerting sur email ou Slack
- definir quelques SLI/SLO simples

Alertes prioritaires recommandees :

- service `down`
- taux d'erreur HTTP > seuil
- p95 de latence trop eleve sur le gateway
- saturation memoire JVM
- backlog ou erreurs sur les consommateurs Kafka
- indisponibilite PostgreSQL
- echec recurrent du scheduler d'outbox

SLI/SLO de depart :

- disponibilite du gateway
- p95 de latence des endpoints critiques
- taux de succes du paiement
- delai entre creation d'un evenement outbox et publication effective

## Plan de modifications dans le code

### 1. Dependencies Maven

Ajouter au minimum dans les services backend :

- `spring-boot-starter-actuator` si absent
- `io.micrometer:micrometer-registry-prometheus`

Puis en phase traces :

- dependances Micrometer Tracing / OTLP compatibles avec la BOM du projet

Remarque :

- il faut verifier la version Spring Boot reelle portee par `ask-bom` avant d'ajouter les artefacts de tracing, pour rester sur un set de versions coherent

### 2. Configuration Spring

Uniformiser la config `management` de tous les services :

- exposer explicitement `health`, `info`, `metrics`, `prometheus`
- garder `readiness` et `liveness`
- ajouter des tags communs :
  - `application`
  - `environment`

Je recommande aussi :

- d'activer les histogrammes de latence sur les endpoints HTTP critiques
- de definir des percentiles `p50`, `p95`, `p99`
- de ne pas exposer `shutdown`

### 3. Instrumentation metier

L'observabilite standard ne suffira pas pour les incidents metier. Il faut ajouter quelques metriques custom :

- compteur des paiements crees
- compteur des paiements echoues
- compteur des emails envoyes / en erreur
- taille du backlog outbox
- nombre de messages consommes / rejetes
- latence de publication outbox

Et quelques spans metier :

- `payment.create`
- `payment.callback.monetico`
- `outbox.publish`
- `notification.send-email`
- `auth.login`

### 4. Logs

Standardiser le pattern ou l'encoder sur tous les services.

Je recommande :

- JSON en environnement `dev` et `prod`
- correlation `traceId` / `spanId`
- un niveau `INFO` sobre
- `DEBUG` reserve au local

### 5. Docker Compose

Ajouter un environnement compose d'observabilite, par exemple :

- `docker-compose/observability/docker-compose.yml`

Il pourrait contenir :

- `grafana`
- `prometheus`
- `loki`
- `tempo`
- `alloy`

Deux approches possibles :

- soit une stack observabilite separee qui observe l'environnement `dev`
- soit une extension du compose `dev`

Je prefere la premiere approche :

- plus lisible
- plus simple a activer/desactiver
- moins risquee pour les services metier

## Ordre d'implementation recommande

1. Metriques Prometheus + Grafana
2. Dashboards de base
3. Logs centralises Loki + Alloy
4. Traces distribuees Tempo + OTLP
5. Alertes
6. Metriques et spans metier

Cet ordre est important :

- si tu commences par les traces, tu augmentes fortement la complexite
- si tu commences par les logs seuls, tu verras les symptomes sans avoir de vision globale
- les metriques sont le meilleur premier investissement

## Priorites specifiques a ton projet

### Priorite haute

- `gateway`
- `payment-service`
- `user-service`

Raison :

- ce sont les composants les plus critiques pour l'experience utilisateur et le chiffre d'affaire

### Priorite moyenne

- `notification-service`
- `formula-service`

### Priorite basse

- `config-server`
- `eureka-server`

Ils doivent etre surveilles, mais ce ne sont pas les premiers a enrichir avec de la metrique metier.

## Risques et points d'attention

- `management.endpoints.web.exposure.include: "*"` est trop large pour un environnement expose
- l'observabilite ne doit pas reveler de donnees sensibles
- les logs structures peuvent faire grossir les volumes rapidement
- les traces doivent etre echantillonnees pour contenir les couts et la volumetrie
- Kafka, RabbitMQ et PostgreSQL auront besoin de dashboards dedies si tu veux une observabilite vraiment utile en production

## Definition de done

Le chantier sera sur de bons rails quand tu pourras :

- voir l'etat de tous les services dans Grafana
- identifier en moins de 5 minutes quel service cause une erreur 5xx
- retrouver les logs d'une requete par `traceId`
- suivre un paiement ou un login sur un trace unique
- recevoir une alerte avant que les utilisateurs remontent le probleme

## Proposition de roadmap courte

### Sprint 1

- ajouter Prometheus + Grafana
- exposer `/actuator/prometheus`
- creer 3 dashboards de base

### Sprint 2

- ajouter Loki + Alloy
- passer les logs en JSON
- correler logs et traces futures

### Sprint 3

- ajouter Tempo
- activer Micrometer Tracing
- tracer les parcours critiques

### Sprint 4

- ajouter alertes
- ajouter metriques metier
- formaliser SLO et runbooks

## Mon avis tranche

Si tu veux une trajectoire efficace, ne cherche pas a tout faire d'un coup.

Commence par :

- `Prometheus`
- `Grafana`
- `micrometer-registry-prometheus`

Puis ajoute :

- `Loki`
- `Grafana Alloy`

Et seulement ensuite :

- `Tempo`
- tracing distribue

Pour ce depot precis, la meilleure premiere livraison n'est pas "Grafana tout seul", mais "Grafana + Prometheus sur tous les services backend avec dashboards gateway/payment/user-service".

## References officielles

- Spring Boot observability: https://docs.spring.io/spring-boot/reference/actuator/observability.html
- Spring Boot metrics: https://docs.spring.io/spring-boot/reference/actuator/metrics.html
- Micrometer Prometheus: https://docs.micrometer.io/micrometer/reference/implementations/prometheus.html
- Micrometer Tracing: https://docs.micrometer.io/tracing/reference/
- Grafana Tempo: https://grafana.com/docs/tempo/latest/getting-started/
- Grafana Loki avec Alloy: https://grafana.com/docs/loki/latest/send-data/alloy/
- Promtail deprecation: https://grafana.com/docs/loki/latest/send-data/promtail/
