---
name: dips-check-service-health
description: Check whether the DIPS Federation Service is up and what version it is running, using the two anonymous status endpoints — the only runtime signal DIPS publishes.
api: DIPS Federation Service
base_url: https://api.dips.no/dips.oauth
operations:
  - get-status-ping
  - health
  - openid-configuration
  - getjwks
generated: '2026-09-02'
method: generated
source: openapi/dips-federation-service-openapi.yml
---

# Check DIPS Federation Service health

DIPS runs no status page. `status.dips.no` and `status.dips.com` do not resolve and
`www.dips.com/status` returns 404. These endpoints are the substitute, and they answer anonymously.

## Steps

1. **Liveness.** `GET /status/ping` (`get-status-ping`). Returns `text/plain` `OK`. Cheap; use it
   for a loop.

2. **Depth.** `GET /status/health` (`health`). Returns a structured document with an overall
   `status` and `statuscode`, plus per-check results:
   - `product.version` — the running build, e.g. `DIPS.Federation.IdentityServer 3.1.55.165`
   - `memory.oom` and `memory.pressure`
   - `database.connection`
   - `.well-known/openid-configuration` — reachability of the *external* identity providers DIPS
     federates with
   - `redis`
   Treat a healthy overall status with a degraded external-IdP check as a partial outage: your own
   login flow may still fail while DIPS reports up.

3. **Configuration drift.** `GET /.well-known/openid-configuration` (`openid-configuration`) and
   `GET /.well-known/openid-configuration/jwks` (`getjwks`). Both anonymous. Diff them against the
   copies in `well-known/` to catch a scope, grant or signing-key change — DIPS publishes no
   changelog, so a diff of these two documents is the only change notification available.

## Rules

- No authentication and no subscription key is needed for `/status/ping`,
  `/.well-known/openid-configuration` or its `jwks`. Everything else on `api.dips.no` needs the key.
- No rate limits are published and no `RateLimit-*` headers come back. Poll conservatively.
- The health document names an internal hostname and a database instance. Do not log or surface it
  to end users.
