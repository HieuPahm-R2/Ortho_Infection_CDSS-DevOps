# Execution Plan: QR Mobile Upload Network Environments

Date: 2026-08-01

## Status

Completed

## Outcome

One explicit LAN IP enables local phone testing of frontend, backend, and
MinIO; production deploys only with public frontend and MinIO origins and keeps
upload-session SSE alive.

## Context

- `docker-buildlocal.yml`
- `docker/docker-compose.yml`
- `.env.local.example`
- `Caddyfile.prod`

## Scope

In scope:

- Bind the local MinIO API to an explicitly configured LAN interface while the
  backend remains loopback-only behind the Vite proxy.
- Feed LAN-reachable frontend, backend CORS, and MinIO URLs to the backend.
- Require production public URLs, configure MinIO CORS, and proxy upload SSE.

Out of scope:

- Router port forwarding or exposure outside the trusted local network.
- Changing Cloudflare Tunnel or DNS records.

## Approach

Use `DEV_LAN_IP` as the single local network input for browser-facing MinIO,
with a loopback default. Keep the backend on loopback behind Vite's same-origin
proxy. Require production public origins at Compose interpolation time and add
the upload event stream to Caddy's unbuffered SSE route.

## Risks And Recovery

- Local ports become reachable on the selected LAN interface; keep the default
  on loopback and document trusted-network use.
- A missing production value will stop deployment before containers start;
  populate the documented environment values or revert the required guards.

## Progress

- [x] Update local Compose and the environment example.
- [x] Harden production Compose and Caddy routing.
- [x] Validate both Compose models and the Caddy configuration.

## Decisions

- 2026-08-01: `108pog.site` and `minio.108pog.site` are the existing production
  authorities.
- 2026-08-01: LAN services bind only to the explicitly selected workstation IP,
  preserving loopback-only behavior by default.
- 2026-08-01: The backend remains loopback-only; Vite owns local LAN API access
  through its same-origin proxy to prevent login CORS failures.

## Validation

- Focused proof: resolved local Compose with `DEV_LAN_IP=192.0.2.10` and
  confirmed frontend, backend, MinIO, and CORS values.
- Integration or end-to-end proof: `caddy validate` reported a valid production
  configuration with the upload-session SSE matcher.
- Repository-required checks: local and production `docker compose config`
  passed with their documented environment values.

## Result

The local backend remains loopback-only behind Vite. MinIO is loopback-only by
default and binds to exactly the configured `DEV_LAN_IP` for trusted-LAN phone
tests. Production requires public frontend and MinIO origins, applies explicit
MinIO CORS, permits same-origin camera access, and streams upload-session events
without the generic API timeout. The production `.env` must contain the three
newly documented public origin values before the next deployment.
