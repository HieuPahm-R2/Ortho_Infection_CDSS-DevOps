# Execution Plan: Local Application Services and Observability

Date: 2026-07-26

## Status

Completed

## Outcome

Run the application services that observability expects inside the local Docker
Compose stack, then verify that Prometheus, Alloy/Loki, OTel Collector/Jaeger,
and Grafana can observe them.

## Context

- `docker-buildlocal.yml` owns local service topology, credentials, ports, and
  container dependencies.
- `docker/observability/prometheus/prometheus.local.yml` owns local scrape
  targets.
- Application Dockerfiles and existing production service definitions provide
  build commands, health endpoints, and OTLP endpoint conventions.

## Scope

In scope:

- Add a locally built Spring Boot backend.
- Start the locally built RAG service with the shared API key.
- Enable the Extract Images API and worker.
- Add PostgreSQL exporter metrics.
- Point local Prometheus at the backend and PostgreSQL exporter containers.
- Recreate the local stack so Alloy receives `DEPLOYMENT_ENV=local`.

Out of scope:

- Production Compose changes.
- Frontend containerization.
- Destructive volume recreation or data migration.

## Approach

1. Reuse the existing application Dockerfiles and production health/OTLP
   conventions.
2. Add local-only credentials and host port bindings in `docker-buildlocal.yml`.
3. Add container-network scrape targets for Spring and PostgreSQL.
4. Build affected images, recreate the stack without deleting named volumes,
   and validate each telemetry path.
5. Make RAG wait for a healthy backend so Spring declares RabbitMQ queues before
   the RAG consumer starts.

## Risks And Recovery

- Risk: application ports conflict with terminal-launched services. Mitigation:
  verified ports 8000, 8002, and 8085 were free before recreation.
- Risk: RAG consumer starts before Spring declares its queue. Mitigation: RAG
  depends on a healthy backend.
- Recovery: stop only the added services with
  `docker compose -f docker-buildlocal.yml stop pji-backend rag-service extract-api extract-worker postgres-exporter`.
- Existing database, object-storage, queue, Loki, Prometheus, Jaeger, and Grafana
  volumes remain intact.

## Progress

- [x] Inspect local build, environment, ports, and dependencies.
- [x] Add Backend, Extract API/Worker, and PostgreSQL exporter.
- [x] Correct local Prometheus scrape configuration.
- [x] Build images and recreate the local stack.
- [x] Validate health, queues, metrics, logs, and traces.

## Decisions

- 2026-07-26: Use container-to-container DNS names locally so observability
  matches production behavior.
- 2026-07-26: Preserve named volumes and recreate containers only.
- 2026-07-26: Order RAG after Backend health because Spring owns declaration of
  `pji.ai.recommendation.queue`.

## Validation

- Focused proof: Compose rendered successfully with 16 services; Prometheus
  configuration passed `promtool check config`; all four application images
  built successfully.
- Integration proof: Backend, RAG, and Extract API health endpoints passed;
  RabbitMQ showed one consumer on both `pji.ai.recommendation.queue` and
  `image_processing`.
- Observability proof: no Prometheus target was down; Backend and PostgreSQL
  exporter were `up=1`; Loki received `env=local` streams for Backend, RAG,
  Extract API/Worker, and PostgreSQL; Jaeger received recent traces from
  Backend, RAG, and Extract API.
- Repository checks: `git diff --check -- docker-buildlocal.yml` passed;
  GitNexus reported low risk with no affected code process.

## Result

The complete local application and observability stack is running. PostgreSQL
exporter emits a harmless warning about its optional `postgres_exporter.yml`
file, but `pg_up=1` and the scrape target is healthy. Alloy also logs its
expected no-op remote configuration registration message; local log forwarding
is verified.
