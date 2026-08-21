---
name: genialis-authenticate
description: >-
  Establish and verify a session against the Genialis Expressions API — what
  anonymous access reaches, how the session cookie works, and how the browser/SAML
  login flows are shaped.
api: Genialis Expressions API
base_url: https://app.genialis.com
spec: openapi/genialis-base-openapi.yaml
generated: '2026-08-21'
method: generated
source: >-
  Grounded in operationIds verified against openapi/genialis-base-openapi.yaml,
  https://docs.genialis.com/resdk/start.html, and live probes on 2026-08-21.
operations:
  - rest_auth_login_create
  - rest_auth_logout_create
  - rest_auth_user_retrieve
  - saml_auth_api_login_create
  - saml_auth_remote_login_auth_id_retrieve
  - saml_auth_remote_login_poll_retrieve
  - about_versions_retrieve
  - about_resdk_minimal_supported_version_retrieve
---

# Authenticating to Genialis Expressions

## What you get without credentials

Public and community data is readable anonymously. Verified 2026-08-21:

```
GET https://app.genialis.com/api/data?limit=1        -> 200, {"count": 29123, ...}
GET https://app.genialis.com/api/collection?limit=1  -> 200, {"count": 107, ...}
GET https://app.genialis.com/api/user                -> 200, []      (empty when anonymous)
```

If your task only touches public datasets, **do not authenticate**. Try the read first.

## The auth model

There is exactly one security scheme in the contract:

```yaml
cookieAuth:
  type: apiKey
  in: cookie
  name: sessionid
```

That is it. **No bearer token. No API key. No OAuth scopes for the API.** Anything that claims a
Genialis API key does not match the published contract.

Human sign-in federates through the Genialis Auth0 tenant — `https://genialis.us.auth0.com/`, which
serves a full OIDC discovery document (PKCE S256, RS256 id_tokens). Auth0 issues the identity; the
Django backend converts it into the `sessionid` cookie the API actually reads. **The API does not
accept an Auth0 access token.**

## Path A — the supported route: resdk

```python
import resdk

res = resdk.Resolwe(url='https://app.genialis.com')
res.login()          # opens an interactive browser login
resdk.start_logging()
```

`res.login()` opens a browser. Omit it and you are the anonymous user. This is the only flow
Genialis documents (<https://docs.genialis.com/resdk/start.html>), and it is not headless — an
unattended agent cannot complete it without a human at a browser.

## Path B — the endpoints in the contract

- `rest_auth_login_create` — `POST /rest-auth/login/` — "Attempt to perform automatic login."
- `rest_auth_logout_create` — `POST /rest-auth/logout/` — Django logout; deletes the Token object.
- `rest_auth_user_retrieve` — `GET /rest-auth/user/` — the current user. **Use this to verify a
  session**: it is the cheapest confirmation that your cookie is live.
- `rest_auth_password_change_create`, `rest_auth_password_reset_create`,
  `rest_auth_password_reset_confirm_create` — password lifecycle.

The published spec declares **no request body schema and no error responses** for any of these, so
the credential fields are not discoverable from the contract. Do not guess them — use `resdk`.

## Path C — enterprise SSO (SAML)

- `saml_auth_api_login_create` — `POST /saml-auth/api-login/`
- `saml_auth_remote_login_auth_id_retrieve` — `GET /saml-auth/remote-login/auth-id/` — "Generate a
  cryptographically secure auth_id token."
- `saml_auth_remote_login_poll_retrieve` — `GET /saml-auth/remote-login/poll/` — "Poll the redis
  server for authentication data."

This is the device-style pairing flow behind the browser login: get an `auth_id`, send the user to
the IdP, poll until the session materialises.

## Compatibility check

`about_resdk_minimal_supported_version_retrieve` — `GET /about/resdk_minimal_supported_version`
returns the minimum client version the server will accept. Check it before a long run; it is the
only deprecation signal this platform emits.

`about_versions_retrieve` — `GET /about/versions` reports the running platform
(2026-08-21: `{"resolwe":"45.1.0","resolwe-bio":"65.0.0","genialis-bio":"63.0.0"}`).

Both live at the **host root**, not under `/api`. Requesting `/api/about/versions` returns the SPA
shell with a 200.

## Failure handling

The spec documents no 401 or 403 anywhere. Expect the Django REST Framework defaults —
`{"detail": "Authentication credentials were not provided."}` and
`{"detail": "You do not have permission to perform this action."}` — and treat any 200 whose
`content-type` is `text/html` as a routing failure, not a success.
