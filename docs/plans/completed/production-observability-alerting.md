# Execution Plan: Production Observability Alerting

Date: 2026-07-26

## Status

Completed

## Outcome

Production observability configuration is tracked in Git, Prometheus evaluates a
baseline set of operational alerts, Alertmanager sends firing and resolved
notifications to Discord without storing the webhook in Git, Grafana provisions
a production overview dashboard, and Jenkins rejects invalid or unhealthy
observability deployments.

## Context

- `docker/docker-compose.yml` owns the production observability services.
- `docker/observability/` owns Prometheus, Alertmanager, Grafana, Loki, Alloy,
  Jaeger, and OTel Collector configuration.
- `Jenkinsfile` owns deploy-time validation and smoke tests.
- The repository has no existing SLO or on-call threshold authority.

## Scope

In scope:

- Track all observability configuration.
- Add baseline availability, backend error-rate, backend average-latency, and
  Prometheus rule-evaluation alerts.
- Deliver alerts through Discord using a secret file.
- Provision a production overview dashboard.
- Validate configurations in CI and smoke-test all observability services.

Out of scope:

- Defining business SLOs.
- Provisioning the Discord webhook itself.
- Adding application metrics to RAG or Extract Images.
- High-availability or remote observability storage.

## Approach

1. Add Prometheus rules and Alertmanager routing to Discord.
2. Add Alertmanager to production Compose and provision a Grafana dashboard.
3. Add static config validation and production smoke tests to Jenkins.
4. Run Compose, Prometheus, Alertmanager, OTel, dashboard, and repository checks.

## Risks And Recovery

- A missing Discord secret must fail deployment before Alertmanager starts.
- Baseline thresholds can be tuned in the versioned rule file once SLOs exist.
- Roll back by reverting the Compose, Prometheus, Alertmanager, Grafana, and
  Jenkins changes; persistent telemetry volumes are not removed.

## Progress

- [x] Track existing observability configuration.
- [x] Add Prometheus rules and Discord Alertmanager configuration.
- [x] Provision the Grafana production overview dashboard.
- [x] Add Compose and Jenkins integration.
- [x] Validate the complete result.

## Decisions

- 2026-07-26: Use Alertmanager's native Discord receiver and
  `webhook_url_file`; never store the webhook URL in Git or environment-backed
  rendered configuration.
- 2026-07-26: Use the requested baseline thresholds as task-local defaults:
  target down for 2 minutes, backend 5xx rate above 5% for 10 minutes, and
  backend average latency above 2 seconds for 10 minutes. The backend currently
  does not publish histogram buckets, so p95 cannot be computed safely.

## Validation

- Focused proof: production and local Compose rendered successfully;
  `promtool check config`, `promtool test rules`, `amtool check-config`, OTel
  Collector validation, dashboard JSON parsing, and repository diff checks
  passed.
- Integration proof: recreated local Prometheus and Grafana; Prometheus loaded
  all four alert rules with successful PromQL evaluation, and Grafana's API
  returned the provisioned `pji-production-overview` dashboard in the
  `PJI Production` folder.
- Deployment proof: Jenkins now validates every observability artifact before
  deploy, validates the production secret path before container recreation, and
  smoke-tests all observability containers plus loaded Prometheus rules.
- Limitation: actual Discord delivery requires the operator-owned webhook secret
  on the production host and the documented manual delivery test after deploy.

## Result

Observability configuration and dashboard/rule artifacts are versioned.
Prometheus evaluates baseline operational alerts, production Alertmanager reads
the Discord webhook from a Docker secret file, and Grafana provisions a
production overview dashboard. Jenkins rejects invalid observability
configuration and unhealthy monitoring services. No webhook secret is stored in
Git.
