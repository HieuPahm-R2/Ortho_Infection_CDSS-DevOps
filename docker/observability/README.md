# Production Observability

Prometheus evaluates the rules in `prometheus/rules/` and sends alerts to
Alertmanager. Alertmanager delivers firing and resolved notifications through a
Discord channel webhook. Grafana loads its datasources and dashboards from
`grafana/provisioning/`.

## Production security boundary

- Caddy's host port is bound to `127.0.0.1:80`. The separately managed
  cloudflared container reaches Caddy at `http://caddy:80` through the dedicated
  external `cloudflare-net` Docker network. Caddy only accepts the documented
  application, Grafana, and MinIO hostnames.
- Alloy has no direct Docker socket mount. It reaches an internal-only Docker
  Socket Proxy which allows container discovery/log reads and denies all POST
  requests.
- Alloy drops logs containing known clinical-payload keys or lines larger than
  16 KB. Common credentials, bearer tokens, JWTs, credential-bearing URLs, and
  Discord webhook URLs are masked before Loki storage.
- The OTel Collector masks sensitive trace attributes and values before export.
  Tail sampling retains errors, traces slower than 2 seconds, and a 10% baseline
  sample of remaining traces.

Redaction is defense in depth. Application services must still avoid logging
patient records, request/response bodies, credentials, or raw prompts.

### Cloudflare deployment prerequisite

Before the first Jenkins deployment containing this hardening:

1. Create the shared network if it does not already exist:

```bash
docker network create cloudflare-net
```

Jenkins also creates it idempotently, but creating it first lets cloudflared be
migrated before Caddy is redeployed.

2. Add `cloudflare-net` to the cloudflared container's Compose configuration or
   `docker run --network` arguments. A manual `docker network connect` is useful
   for testing but is not durable across container recreation.
3. Recreate cloudflared and confirm both containers join `cloudflare-net`.
4. In the Cloudflare dashboard, change every Caddy-routed Published Application
   service from the server IP to:

```text
http://caddy:80
```

The hostname `caddy` is resolved by Docker DNS on `cloudflare-net`. Do not use
`127.0.0.1` here because cloudflared runs in a separate container, and do not
keep the server's public IP because that requires exposing the origin port.

## Discord webhook secret

Do not put the webhook URL in Git or directly in Compose. On the production
server:

```bash
install -d -m 700 /opt/pji-advisor/secrets
printf '%s' 'https://discord.com/api/webhooks/REPLACE_ME' \
  > /opt/pji-advisor/secrets/discord_webhook_url
chmod 600 /opt/pji-advisor/secrets/discord_webhook_url
```

Set this path in `/opt/pji-advisor/.env`:

```dotenv
DISCORD_WEBHOOK_URL_FILE=/opt/pji-advisor/secrets/discord_webhook_url
```

Compose mounts the file at `/run/secrets/discord_webhook_url` inside
Alertmanager. The webhook is never stored in the repository or rendered
Alertmanager configuration.

## Baseline alerts

- Any configured Prometheus target down for 2 minutes.
- Backend 5xx ratio above 5% for 10 minutes.
- Backend average request latency above 2 seconds for 10 minutes.
- Prometheus rule evaluation failures.

These are operational baselines, not business SLOs. Update the versioned rule
file when product SLOs are approved.

## Manual delivery test

After deployment, submit a synthetic alert to Alertmanager:

```bash
curl -fsS -X POST http://127.0.0.1:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[{"labels":{"alertname":"DiscordDeliveryTest","severity":"warning","job":"manual"},"annotations":{"summary":"Discord delivery test","description":"Synthetic test from the production host"}}]'
```

Resolve it after Discord receives the notification:

```bash
curl -fsS -X POST http://127.0.0.1:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[{"labels":{"alertname":"DiscordDeliveryTest","severity":"warning","job":"manual"},"annotations":{"summary":"Discord delivery test","description":"Synthetic test from the production host"},"endsAt":"2000-01-01T00:00:00Z"}]'
```
