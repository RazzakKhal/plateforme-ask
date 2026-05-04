# Observability Sprint 1

## Objectif du sprint

Le sprint 1 doit livrer une premiere observabilite exploitable, sans entrer encore dans les logs centralises ni dans le tracing distribue.

Le resultat attendu est simple :

- chaque backend expose `/actuator/prometheus`
- `Prometheus` collecte les metriques de tous les services utiles
- `Grafana` affiche 3 dashboards de base pour piloter la plateforme

Ce sprint sert a repondre rapidement a trois questions :

- quel service est en train de tomber
- quel service devient lent
- quel service consomme anormalement la JVM

## Ce que l'on ajoute et a quoi cela sert

### 1. `micrometer-registry-prometheus`

But :

- exposer les metriques Spring Boot au format que Prometheus sait scraper

Pourquoi c'est necessaire :

- `spring-boot-starter-actuator` expose deja les endpoints de supervision
- mais sans le registre Prometheus, tu n'as pas l'endpoint `/actuator/prometheus`
- c'est donc la brique qui transforme les metriques internes Spring/Micrometer en sortie exploitable par Prometheus

### 2. `Prometheus`

But :

- collecter periodiquement les metriques de tous les services

Pourquoi c'est necessaire :

- Grafana n'est pas un collecteur de metriques
- il faut un stockage et un moteur de requetes temporelles
- Prometheus va interroger chaque service toutes les X secondes et stocker l'historique

### 3. `Grafana`

But :

- visualiser les metriques dans des dashboards lisibles
- centraliser l'analyse et, plus tard, les alertes

Pourquoi c'est necessaire :

- lire du Prometheus brut est peu pratique pour l'exploitation
- Grafana permet de transformer les metriques en panneaux utiles pour l'equipe

### 4. 3 dashboards de base

But :

- avoir des vues directement actionnables au lieu d'une simple collecte technique

Pourquoi c'est necessaire :

- la collecte seule ne donne pas de valeur si personne ne voit rapidement les signaux utiles
- ces dashboards serviront de socle avant d'ajouter logs, traces et alerting

## Perimetre de sprint 1

Services a inclure dans la supervision :

- `gateway`
- `user-service`
- `formula-service`
- `payment-service`
- `notification-service`

Services optionnels dans ce sprint :

- `config-server`
- `eureka-server`

Pourquoi ce perimetre :

- les 5 premiers portent les flux applicatifs utiles
- `config-server` et `eureka-server` sont importants, mais moins prioritaires pour la premiere valeur visible

Le frontend Angular n'entre pas dans ce sprint.

## Etat actuel du depot

Ce depot a deja plusieurs pre-requis utiles :

- Spring Boot `3.4.1` est porte par `ask-bom`
- les services backend exposent deja `Actuator`
- les healthchecks Docker utilisent deja `/actuator/health/readiness`
- les fichiers `application.yml` exposent actuellement `management.endpoints.web.exposure.include: "*"`

Conclusion :

- la base de supervision existe deja
- il faut maintenant ajouter le registre Prometheus, resserrer l'exposition des endpoints et brancher une stack `Prometheus + Grafana`

## Plan d'implementation

### Etape 1 - Ajouter le registre Prometheus dans chaque backend

Services a modifier :

- `gateway/pom.xml`
- `user-service/pom.xml`
- `plateforme-formula-service/pom.xml`
- `plateforme-payment-service/pom.xml`
- `plateforme-notification-service/pom.xml`
- optionnellement `plateforme-config-server/pom.xml`
- optionnellement `plateforme-eureka-server/pom.xml`

Dependance a ajouter :

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Pourquoi cet ajout :

- Spring Boot 3.4.1 gere deja tres bien Micrometer
- aucune version manuelle n'est a fixer si la BOM Spring Boot la gere
- l'ajout est leger et n'impacte pas le code metier

Effet attendu :

- `/actuator/prometheus` devient disponible sur chaque service concerne

## Etape 2 - Exposer proprement `/actuator/prometheus`

Aujourd'hui, vos services exposent trop largement les endpoints d'administration avec `include: "*"`.

Pour le sprint 1, il faut uniformiser la configuration `management` afin de n'exposer que ce qui sert a la supervision de base.

Configuration cible recommandee :

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
  info:
    env:
      enabled: true
  health:
    probes:
      enabled: true
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
```

Pourquoi chaque partie existe :

- `health` : permet de garder les checks de disponibilite
- `info` : permet d'afficher un minimum d'information applicative
- `metrics` : permet les metriques Actuator detaillees
- `prometheus` : permet a Prometheus de scraper les metriques
- `application` dans les tags : facilite les dashboards multi-services

Point important :

- il faut supprimer l'exposition `shutdown` en acces libre
- il faut eviter `include: "*"` en dehors d'un besoin temporaire tres controle

## Etape 3 - Ajouter Prometheus dans Docker Compose

Je recommande de creer un environnement dedie :

- `docker-compose/observability/docker-compose.yml`

Pourquoi un compose dedie :

- la stack d'observabilite reste isolee
- elle peut etre montee ou arretee sans toucher aux services metier
- le dossier reste propre a mesure que Grafana, Loki et Tempo arriveront plus tard

Services a definir dans ce compose :

- `prometheus`
- `grafana`

Volumes a prevoir :

- persistance des donnees Prometheus
- persistance de la configuration Grafana

Ports conseilles :

- `9090` pour Prometheus
- `3000` pour Grafana

Exemple de structure :

```text
docker-compose/
  observability/
    docker-compose.yml
    prometheus/
      prometheus.yml
    grafana/
      provisioning/
        datasources/
          prometheus.yml
        dashboards/
          dashboards.yml
          platform-overview.json
          http-performance.json
          jvm-resources.json
```

## Etape 4 - Configurer Prometheus

Fichier a creer :

- `docker-compose/observability/prometheus/prometheus.yml`

Role de ce fichier :

- dire a Prometheus quels services scraper
- definir la frequence de collecte

Configuration de depart recommandee :

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: gateway
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['gateway:8072']

  - job_name: user-service
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['plateforme-user-service:3001']

  - job_name: formula-service
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['plateforme-formula-service:3002']

  - job_name: payment-service
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['plateforme-payment-service:3003']

  - job_name: notification-service
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['plateforme-notification-service:3004']
```

Pourquoi cette configuration :

- un `job_name` par service simplifie les filtres et dashboards
- `metrics_path` est explicite
- `15s` donne une lecture suffisante pour un environnement de dev sans surcharger inutilement

Point d'attention :

- les cibles doivent utiliser les noms reseau Docker reels, pas `localhost`
- si le compose observability partage le meme reseau que `docker-compose/dev`, il verra les containers par leur nom de service ou `container_name`

## Etape 5 - Configurer Grafana

Grafana doit etre provisionne des le depart, pour eviter une configuration manuelle fragile.

### Datasource

Fichier conseille :

- `docker-compose/observability/grafana/provisioning/datasources/prometheus.yml`

Role :

- brancher automatiquement Grafana sur Prometheus

Exemple :

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

Pourquoi le provisionning est utile :

- l'environnement est reproductible
- tu n'as pas besoin de cliquer dans l'UI a chaque recreation de stack

### Dashboards

Fichiers conseilles :

- `docker-compose/observability/grafana/provisioning/dashboards/dashboards.yml`
- `docker-compose/observability/grafana/provisioning/dashboards/*.json`

Role :

- charger automatiquement les dashboards a l'ouverture de Grafana

## Les 3 dashboards de base

### Dashboard 1 - Platform Overview

Objectif :

- avoir une vue synthese de l'etat des services

Ce que le dashboard doit montrer :

- statut `UP/DOWN` par service
- nombre d'instances visibles
- taux de requetes par service
- taux d'erreur par service

Pourquoi ce dashboard existe :

- il repond a la question "quel service pose probleme maintenant"
- c'est la vue d'entree de l'equipe quand un incident survient

Panneaux recommandes :

- table ou stat panel par service avec `up`
- requetes par seconde par service
- erreurs HTTP 5xx par service
- total des requetes sur le gateway

Exemples de metriques utiles :

- `up`
- `http_server_requests_seconds_count`
- `http_server_requests_seconds_sum`

### Dashboard 2 - HTTP Performance

Objectif :

- suivre les performances API

Ce que le dashboard doit montrer :

- debit de requetes
- latence moyenne
- p95 de latence
- repartition 2xx / 4xx / 5xx
- focus sur `gateway`, `user-service` et `payment-service`

Pourquoi ce dashboard existe :

- une API peut etre `UP` tout en etant inutilisable car trop lente
- c'est le dashboard qui permet de voir si la degradation vient du trafic ou du temps de traitement

Panneaux recommandes :

- requetes par seconde par service
- latence moyenne par service
- p95 global pour les endpoints critiques
- compteur d'erreurs 4xx et 5xx

Metriques utiles :

- `rate(http_server_requests_seconds_count[5m])`
- `rate(http_server_requests_seconds_sum[5m])`
- `histogram_quantile(0.95, ...)` si les buckets sont actifs

Point important :

- si tu veux un vrai `p95`, il faut activer les histogrammes de latence sur les services HTTP

Configuration utile a ajouter :

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
```

Pourquoi cet ajout :

- sans histogrammes, Prometheus ne pourra pas calculer proprement les quantiles de latence

### Dashboard 3 - JVM Resources

Objectif :

- surveiller la sante technique de chaque service Java

Ce que le dashboard doit montrer :

- memoire heap et non-heap
- threads actifs
- activite GC
- CPU process
- uptime

Pourquoi ce dashboard existe :

- beaucoup d'incidents Java apparaissent d'abord comme un probleme de memoire, threads ou GC
- il permet de distinguer un probleme applicatif d'un probleme de runtime

Panneaux recommandes :

- heap utilisee par service
- nombre de threads vivants
- temps GC
- CPU process
- uptime par service

Metriques utiles :

- `jvm_memory_used_bytes`
- `jvm_memory_max_bytes`
- `jvm_threads_live_threads`
- `jvm_gc_pause_seconds_count`
- `process_cpu_usage`
- `process_uptime_seconds`

## Ordre concret de realisation

1. Ajouter `micrometer-registry-prometheus` dans les `pom.xml`
2. Uniformiser `management` dans les `application.yml` backend
3. Activer les histogrammes HTTP
4. Creer `docker-compose/observability`
5. Ajouter `prometheus.yml`
6. Ajouter le provisionning Grafana
7. Creer les 3 dashboards JSON
8. Verifier que chaque cible est bien scrappee

Pourquoi cet ordre :

- il commence par rendre les services scrapeables
- ensuite seulement il ajoute les outils de collecte et de visualisation
- il evite de construire Grafana sur des metriques absentes ou incoherentes

## Verification attendue a la fin du sprint

### Verification backend

Pour chaque service inclus :

- `GET /actuator/health` repond
- `GET /actuator/prometheus` repond
- la metrique `http_server_requests_seconds_count` est presente

### Verification Prometheus

Dans l'UI Prometheus :

- chaque target apparait `UP`
- les requetes `up` et `http_server_requests_seconds_count` retournent des series

### Verification Grafana

Dans Grafana :

- la datasource Prometheus est chargee automatiquement
- les 3 dashboards sont visibles sans configuration manuelle
- les panneaux affichent des donnees pour au moins `gateway`, `user-service` et `payment-service`

## Limites volontaires du sprint 1

Ce sprint ne doit pas encore traiter :

- logs centralises
- traces distribuees
- alerting
- SLO
- metriques metier custom

Pourquoi on les differe :

- ils ajoutent de la complexite
- ils ne sont utiles qu'une fois la base metriques stable et lisible

## Mon avis sur la meilleure implementation

La meilleure livraison sprint 1 pour ce depot est :

- ajouter `micrometer-registry-prometheus` sur tous les services backend prioritaires
- remplacer `include: "*"` par une exposition precise
- creer un `docker-compose/observability`
- provisionner Grafana des le depart
- livrer exactement 3 dashboards :
  - `Platform Overview`
  - `HTTP Performance`
  - `JVM Resources`

C'est assez petit pour etre livre vite, et assez utile pour servir de base serieuse au sprint 2.
