# Execution Plan: Production Observability Security Hardening

Date: 2026-07-26

## Status

Completed

## Outcome

The production observability stack has a restricted Cloudflare-to-Caddy origin
boundary, Alloy can read Docker logs without direct access to the Docker socket,
and logs/traces are filtered and sampled before storage.

## Context

- `Caddyfile.prod`
- `docker/docker-compose.yml`
- `docker-buildlocal.yml`
- `docker/observability/alloy/config.alloy`
- `docker/observability/otel-collector/config.yml`
- `docker/observability/jaeger/config.yml`
- `Jenkinsfile`

The cloudflared connector and Caddy run as separate Docker containers on the
same production host. Caddy therefore keeps a loopback-only host port and also
joins a dedicated external Docker network where cloudflared reaches it by
service name. The application hostnames in `Caddyfile.prod` are the authority
for the accepted origin `Host` values.

## Scope

In scope:

- Bind the Caddy origin listener to host loopback and remove unused host TLS
  ports.
- Restrict the main Caddy site to the production application hostnames.
- Put a read-only Docker API proxy between Alloy and `/var/run/docker.sock`.
- Redact credentials and sensitive clinical payloads from collected logs.
- Redact sensitive trace attributes and retain errors, slow traces, and a 10%
  baseline sample.
- Validate configs and observable proxy/redaction behavior.

Out of scope:

- Cloudflare dashboard or tunnel credential changes.
- Host firewall management.
- Application-level log/trace instrumentation changes in child repositories.
- Business SLO tuning.

## Approach

1. Lock the host origin port to loopback and reject unknown hostnames.
2. Add an internal-only Docker Socket Proxy permitting read-only container
   discovery/log endpoints, then point Alloy to it.
3. Apply log filtering in Alloy and trace redaction/tail sampling in the OTel
   Collector.
4. Extend CI and smoke checks for the new security boundary.
5. Validate static configuration and exercise the local proxy/log pipeline.

## Risks And Recovery

- A tunnel running on a different host cannot reach a loopback-only origin.
  Repository evidence says the tunnel is on the same host; if deployment
  topology differs, stop deployment and explicitly bind Caddy to the tunnel
  interface instead.
- Over-broad redaction may reduce debugging detail. Credentials and clinical
  payload keys are intentionally prioritized; restore the previous config to
  roll back while rules are tuned.
- Tail sampling adds a short decision delay and memory usage. The collector
  already has a memory limiter; restore the previous trace pipeline if resource
  pressure is observed.
- Rollback is a normal Compose/config redeploy of the previous revision; no data
  migration or destructive operation is involved.

## Progress

- [x] Confirm repository authority and affected topology.
- [x] Restrict the Caddy origin boundary.
- [x] Add Docker Socket Proxy and remove Alloy socket access.
- [x] Add log/trace data minimization.
- [x] Extend CI/smoke checks.
- [x] Run focused and integration validation.

## Decisions

- 2026-07-26: Bind production Caddy's host port to `127.0.0.1:80`.
- 2026-07-28: Connect the separately managed cloudflared container and Caddy
  through external `cloudflare-net`, using `http://caddy:80` as the dashboard
  Service URL. Container loopback cannot cross this boundary.
- 2026-07-26: Keep errors and traces slower than 2 seconds, plus a 10% baseline
  trace sample. This preserves incident evidence while avoiding 100% retention.
- 2026-07-26: Drop logs containing known clinical-payload keys instead of trying
  to partially sanitize arbitrary free-form medical content.

## Validation

- Focused proof: production/local Compose rendered; Caddy and OTel Collector
  validators passed; Alloy formatter/parser passed.
- Integration or end-to-end proof: Docker Socket Proxy GET returned data and
  POST returned 403; Alloy reported all components healthy and forwarded logs;
  the log probe masked credentials and dropped the clinical-payload line; an
  error trace survived tail sampling while its password and URL were masked;
  Caddy returned 421 for an unknown Host and routed an allowed Host.
- Repository-required checks: `git diff --check` passed and GitNexus reported
  low risk with no changed symbols or affected execution flows.

## Result

Implemented all three hardening layers. Production deployment still requires
the operator to attach the cloudflared container to `cloudflare-net` and point
the Cloudflare Tunnel origin at `http://caddy:80` before Jenkins runs. Runtime
validation used only synthetic credentials and traces.

The integration cleanup initially removed stopped local Compose containers.
Their persistent volumes were not deleted, and all affected containers were
recreated in the original non-running (`Created`) state.
