# Repository Guidelines
## Project Structure & Module Organization
The root `pom.xml` orchestrates all Spring Boot microservices: `user-service`, `plateforme-formula-service`, `plateforme-config-server`, `plateforme-eureka-server`, `gateway`, `plateforme-payment-service`, and `plateforme-notification-service`. Each service follows the standard `src/main/java` and `src/test/java` layout with configuration in `src/main/resources`. The Angular client lives in `web/`, with feature modules under `web/src/app`. Docker definitions sit under `docker-compose/` per environment (`dev`, `local`, `prod`); update the matching `.env` before starting stacks.

## Build, Test, and Development Commands
From the repository root run `./mvnw clean verify` to build and unit test every backend module. Use `./mvnw spring-boot:run -pl gateway -am` (swap `gateway` for any service) for focused development with dependencies resolved. Generate container images with `./mvnw jib:dockerBuild` inside a service. For the Angular client, run `npm install` once, then `npm run start -- --configuration development` for local work and `npm run build` to ship optimized assets. Bring up integrated dependencies with `docker compose -f docker-compose/dev/docker-compose.yml up --build`.

## Coding Style & Naming Conventions
Java modules target Java 17 and Spring Boot 3; prefer Lombok annotations for boilerplate and MapStruct for mappings. Maintain four space indentation, one class per file, and camelCase method names. REST controllers expose nouns in plural form (for example `/users`). Angular code inherits `web/.editorconfig`: two space indentation, single quotes, and kebab-case file names. Keep environment specific settings in `application.yml` and avoid hard coding credentials.

## Testing Guidelines
Backend services rely on Spring Boot Test with Jacoco configured; add `*Test` classes under `src/test/java` and ensure new code keeps module coverage above existing levels. Use in-memory fixtures and Kafka test utilities where available. Frontend tests run through `npm run test`, which produces coverage reports in `web/coverage/`; keep high level journeys in Cypress once added. Document manual verification steps in the related service README when integration behaviour changes.

## Commit & Pull Request Guidelines
Recent history shows terse messages; adopt an imperative Conventional Commit format such as `feat(user-service): add password reset audit`. Reference Jira or GitHub issue IDs when relevant. For pull requests, include a short summary, affected modules checklist, test evidence (`./mvnw test`, `npm run test`), and screenshots or curl examples for API changes. Request at least one reviewer from another service team before merge and wait for CI to pass.
