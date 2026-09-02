---
name: dips-authenticate-and-select-role
description: Sign a user into Open DIPS with OpenID Connect and select the DIPS user role that carries their clinical authority, producing a token every other DIPS API will accept.
api: DIPS Federation Service
base_url: https://api.dips.no/dips.oauth
operations:
  - authorize
  - callback
  - token
  - userroles
  - selectuserrole
  - userinfo
generated: '2026-09-02'
method: generated
source: openapi/dips-federation-service-openapi.yml
---

# Authenticate a user and select a DIPS role

Every Open DIPS call needs two credentials at once, and this is the step that produces the second
one. Getting only the first is the most common way to be stuck at HTTP 401.

## Before you start

- Sign up at `https://dips.developer.azure-api.net/signup` and copy the subscription key from your
  profile page. It goes on **every** request to `api.dips.no` as `Ocp-Apim-Subscription-Key`
  (or `?subscription-key=`), including the ones below.
- Sandbox test user: username `opendips`, password `opendips`. The role to pick when prompted is
  the one DIPS marks *Full funksjonstilgang og VID datatilgang (BT)*.
- Everything you touch is synthetic data in a virtual DIPS Arena. Nothing here reaches a real
  patient record.

## Steps

1. **Discover the endpoints.** `GET /.well-known/openid-configuration` (`openid-configuration`).
   Do not hard-code the endpoints — read `authorization_endpoint`, `token_endpoint` and `jwks_uri`
   from this document. It is served anonymously.

2. **Send the user to authorize.** `GET /connect/authorize` (`authorize`) with `client_id`,
   `redirect_uri`, `response_type=code`, `scope`, `state` and a PKCE `code_challenge`.
   `code_challenge_methods_supported` advertises both `S256` and `plain` — use `S256`.
   Ask for `openid profile offline_access` plus the scope for the API you actually need
   (`dips-fhir` for the FHIR API). See `scopes/dips-scopes.yml` for the 49 scopes the issuer
   advertises.

3. **Let the user consent.** `GET /consent` (`consent`) and `POST /consent` (`consent-post`) are the
   DIPS-hosted consent pages. The browser handles these; your code does not call them directly.
   The result comes back to `GET /connect/authorize/callback` (`callback`) and then to your
   `redirect_uri` with `code` and `state`.

4. **Exchange the code for tokens.** `POST /connect/token` (`token`) as
   `application/x-www-form-urlencoded` with `grant_type=authorization_code`, `code`, `redirect_uri`
   and your `code_verifier`. You get back `access_token`, `token_type`, `expires_in`,
   `refresh_token`, `scope` and `id_token`.

5. **List the user's DIPS roles.** `GET /userrole` (`userroles`) with the access token. This step
   has no OAuth equivalent and is easy to miss — a DIPS user acts under a role, and the token you
   hold after step 4 is authenticated but not yet clinically privileged.

6. **Select one.** `POST /userrole/selectuserrole` (`selectuserrole`). The selected role is bound
   into the session; subsequent tokens carry that authority.

7. **Confirm who you are.** `GET /connect/userinfo` (`userinfo`) returns the claims — `sub`, `name`,
   `family_name`, `given_name`, `preferred_username` and `role`.

## Rules

- **Errors** are the OAuth 2.0 envelope (`{"error", "error_description"}`) from the federation
  service, and the Azure gateway envelope (`{"statusCode", "message"}`) from anything in front of
  it. There is no `application/problem+json` on this surface. A 401 reading
  *"Access denied due to missing subscription key"* is the gateway, not DIPS — your subscription
  key is missing, not your token. See `errors/dips-problem-types.yml`.
- **Idempotency:** none published. `POST /connect/token` is single-use by the authorization-code
  grant's own semantics; do not retry a used code, request a fresh one.
- **Undo:** revoke a token with `POST /connect/revocation` (`revocation`); end the session with
  `/connect/endsession` (`endsession`). DIPS states no window for either — see
  `conventions/dips-conventions.yml`.
- **Rate limits:** none published, and no `RateLimit-*` or `Retry-After` headers were observed.
  Back off on your own signal rather than expecting one.
