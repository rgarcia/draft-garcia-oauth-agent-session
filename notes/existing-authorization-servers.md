# Worked example: a consumer site that already runs an authorization server

Captured 2026-10-25 from the public, unauthenticated metadata of a large
consumer travel website. Recorded here as evidence for the claim in the
draft's appendix that for many sites the profile is "configuration of an
existing component and one new route."

## What the site already publishes

`GET https://<accounts host>/.well-known/oauth-authorization-server`

```json
{
  "issuer": "https://<accounts host>",
  "authorization_endpoint": "https://<accounts host>/oauth2/authorize/v3",
  "token_endpoint": "https://<accounts host>/oauth2/token/v2",
  "revocation_endpoint": "https://<accounts host>/oauth2/revoke",
  "jwks_uri": "https://<accounts host>/openid/jwk_set.json",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256", "plain"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"],
  "revocation_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"],
  "scopes_supported": [
    "openid", "profile", "email", "phone", "address", "offline_access",
    "https://<accounts host>/scope/payments",
    "https://<accounts host>/scope/order/read_only",
    "https://<accounts host>/scope/order/read_write",
    "..."
  ]
}
```

The same host also serves `/.well-known/openid-configuration` with
`userinfo_endpoint`, `end_session_endpoint`, and `id_token_signing_alg_values_supported`.

The site's own sign-in page is a first-party OAuth client of this server. Its
page state carries `operation: "authorize"`, a fixed `client_id`, a
`booking_default`-style scope, a `redirect_uri` on the main site that
exchanges the code for a web session, and a `code_challenge` slot. In other
words the site already performs "authorization code to session cookie" for
itself every time a user signs in.

## What is missing relative to the profile

| requirement | status | nature of the work |
|---|---|---|
| `client_id_metadata_document_supported: true` | absent | enable URL client ids; fetch and cache the document |
| `device_authorization_endpoint` | absent | enable RFC 8628 on the existing server |
| `device_code` in `grant_types_supported` | absent | same |
| `none` in `token_endpoint_auth_methods_supported` | absent | allow public clients, scoped to CIMD clients |
| `web_session` in `scopes_supported` | absent | one scope definition plus consent text |
| `web_session_uri` route | absent | one new route: validate bearer token, set the session cookie the first-party flow already sets |

Behavioural confirmation that CIMD is not supported today, from the
authorization endpoint:

```
client_id=https://example.net/agent-client.json  -> 400 invalid_client
client_id=not-a-real-client                      -> 400 invalid_client
client_id=<the site's own first-party id>        -> 400 invalid_request, "Invalid redirect_uri"
```

The URL client id fails at client lookup, before redirect_uri validation, so
the server never attempted to fetch the document.

## Why this matters for the draft

* The hard parts of an authorization server (token endpoint, PKCE, refresh
  tokens, revocation, scoped consent UI) exist. The delta is configuration
  plus one route.
* The site already has a "code to cookie" step for its own login. The
  `web_session_uri` is that step with a bearer token as input instead of an
  authorization code.
* No developer portal or client registration exists. That is the gap CIMD
  fills, and the reason DCR would be a heavier ask.
