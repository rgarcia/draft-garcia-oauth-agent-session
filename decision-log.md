# Decision log

Design decisions for draft-garcia-oauth-agent-session, with the alternatives
considered. Most recent first within each state. Open items are at the bottom.

## Decided

### D8 — Individual submission named for the OAuth WG, hosted on i-d-template (2026-10-25)
The draft is `draft-garcia-oauth-agent-session`: an individual submission
with the `oauth` working group token in the name, which the IETF naming
guidance calls the popular convention for a draft seeking adoption by a
group. The repository name matches the draft name because the
martinthomson/i-d-template tooling derives the editor's copy and build
targets from the markdown filename. Alternatives: `draft-ietf-oauth-*`
(not permitted before adoption); an organization name in the author slot
(allowed by the rules but reads as a vendor specification); omitting the
working group token as draft-carleton-workload-authz-grant does (fine, but
this profile is unambiguously OAuth work).

### D7 — The access token is not single-use and is not revoked with the session (2026-10-25)
The `web_session` access token has a 300 second maximum lifetime and may be
presented more than once within it. Terminating a session does not invalidate
the token that created it. Rationale: a five minute ttl bounds exposure well
enough, and single-use or linked revocation forces the site to keep a
consumed-token store on the web tier, which is exactly the kind of lifecycle
state the profile is trying not to ask for. Alternatives: single-use token
(rejected: adds state for little gain); revoke token on session termination
(rejected: couples two stores that are normally separate).

### D6 — Both GET-with-header and POST-with-form-body are mandatory at `web_session_uri` (2026-10-25)
Sites MUST accept `Authorization: Bearer` on a GET and `access_token` in a
form-encoded POST body. Rationale: agents driving a browser over an automation
protocol can attach a header to a navigation, which is the simplest possible
exchange; agents that can only cause a form submission need the POST form.
Both keep the token out of the URL. The token in a query string was rejected
outright: history, logs, Referer. The POST form carries a login-CSRF risk,
addressed by requiring a same-origin Origin or Sec-Fetch-Site check.

### D5 — `web_session_uri` is a token response parameter, not a metadata field (2026-10-25)
The exchange endpoint is returned alongside the access token rather than
advertised in RFC 8414 metadata. Rationale: RFC 6749 §5.1 already permits
extension parameters in the token response, so no new discovery mechanism is
needed; the site can vary the URL per client or deployment; and the value is
only visible to a client that completed authorization. Alternatives: a new
metadata field (deferred, see open item O1: may be added in addition for
pre-flight discovery); a fixed well-known path (rejected: constrains the
site's routing for no benefit).

### D4 — Discovery is purely RFC 8414 metadata (2026-10-25)
An agent decides whether a site supports the profile from five existing
metadata fields: `client_id_metadata_document_supported`,
`device_authorization_endpoint`, device_code in `grant_types_supported`,
`none` in `token_endpoint_auth_methods_supported`, and `web_session` in
`scopes_supported`. No new well-known document. Rationale: every field is
already standard and already published by servers that support the
underlying specs. Verified against a real consumer site's metadata on
2026-10-25; it published everything except the CIMD flag, the device
endpoint, `none`, and the scope.

### D3 — Device Authorization Grant, not an out-of-band code the user copies back (2026-10-25)
The original sketch had the user approve on their device, receive a one-time
code, and paste it back to the agent, which then redeemed it. Replaced with
RFC 8628. Rationale: the copy-back step is the deprecated `oob` pattern. It
asks the user to carry a secret from the site into an agent chat, puts the
secret in a URL, and gives phishers a natural script. With the device grant
the user types a code into the site's own origin and nothing travels back;
the `device_code` the agent holds binds the result. Also drops the need for
the agent to have a browser open during approval, and makes re-mint possible
via a standard refresh token instead of an invented mechanism.

### D2 — Output is a web session, not an API token (2026-10-25)
The deliverable to the agent is the site's ordinary session cookie, set at a
site-controlled endpoint. The `web_session` access token exists only to
bridge from the authorization server to that cookie. Rationale: the target
adopters are websites without APIs and without plans to build one. A session
reuses their existing session store, active-sessions UI, and risk tooling.
Scoped API tokens would require a resource server and an authorization
model in every route. Alternative: define the access token as usable against
the site's API (rejected for this profile; a site can add that later on the
same authorization server without changing anything here).

### D1 — Client identity is a Client ID Metadata Document URL (2026-10-25)
Agents identify themselves with a CIMD URL rather than a pre-registered
client id or Dynamic Client Registration. Rationale: the site is not a
developer platform and should not need a portal, app review, or per-client
secret storage. CIMD puts client state with the agent platform; the site
fetches and caches it. DCR was rejected because it requires the site to
operate a registration endpoint and hold per-client state. Pre-registration
was rejected because it does not scale past a handful of agent vendors and
reintroduces the portal.

## Open

### O1 — Advertise `web_session_uri` in metadata too?
Would let an agent confirm the exchange endpoint exists before involving the
user. Costs a new metadata field. Leaning yes as an OPTIONAL addition that
does not replace the token response parameter.

### O2 — Agent-requested session lifetime
Whether the device authorization request should carry a requested lifetime
(for example via a parameter or a scope suffix), or whether lifetime stays
entirely site policy shown on the consent screen. Leaning site policy for
the first version.

### O3 — Normative default policy for agent sessions
The recommended denylist in the draft (no credential or factor changes, no
new payment instruments, no account deletion, no further agent approvals) is
a SHOULD-level recommendation. Open whether to make any of it a MUST.

### O4 — Revocation signal to the agent's browser
Today a revoked session simply stops working. Whether to define a response
header or status the site can use so the agent can tell revocation from a
transient failure and decide whether to re-run the flow.

### O5 — Working group fit
Target is the OAuth WG. The profile touches nothing outside OAuth, but the
web-session output and the bot-detection carve-out may draw interest from
the Web Bot Auth effort. Decide after first list feedback.
