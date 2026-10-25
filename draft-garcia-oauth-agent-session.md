---
title: "OAuth 2.0 Agent Session"
abbrev: "Agent Session"
category: info

docname: draft-garcia-oauth-agent-session-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "sec"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - agent
 - delegation
 - device authorization grant
 - client id metadata document
 - web session
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "rgarcia/draft-garcia-oauth-agent-session"
  latest: "https://rgarcia.github.io/draft-garcia-oauth-agent-session/draft-garcia-oauth-agent-session.html"

author:
 -
    fullname: Rafael Garcia
    organization: Kernel
    email: raf@kernel.sh
    role: editor

normative:
  RFC6749:
  RFC6750:
  RFC8414:
  RFC8628:
  RFC6265:
  RFC9110:
  CIMD: I-D.ietf-oauth-client-id-metadata-document

informative:
  RFC7009:
  RFC7591:
  RFC9700:
  RFC8252:
  WBA: I-D.meunier-web-bot-auth-architecture
  OIDC-DISCOVERY:
    title: OpenID Connect Discovery 1.0 incorporating errata set 2
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
    date: 2023-12
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: E. Jay

--- abstract

This document defines Agent Session, a profile of the OAuth 2.0 Device
Authorization Grant and OAuth Client ID Metadata Documents that lets a software
agent obtain an ordinary logged-in web session on a website on behalf of a
user, with the user's consent and without the user disclosing credentials to
the agent. The profile is designed to be implementable by websites that are not
developer platforms: it requires no API, no resource server, and no token
lifecycle beyond what the website's existing authorization server already
provides. It adds two elements to existing specifications: the `web_session`
scope and the `web_session_uri` token response parameter.


--- middle

# Introduction {#introduction}

Software agents increasingly use websites on behalf of the people who direct
them. To do so for a signed-in user, an agent needs to be logged in as that
user. Today this is accomplished in one of three ways, each with a cost that
falls on the wrong party:

* The user shares their password with the agent. The website cannot distinguish
  the agent from the user, the user cannot scope or revoke the agent's access
  without changing their password, and the credential is exposed to the agent
  and its operator.

* The user signs in inside a browser the agent controls, and the agent keeps the
  resulting session. The website still cannot distinguish the agent from the
  user, and the user has to perform the login inside an environment they do not
  control, which is exactly the behaviour phishing training teaches them to
  avoid.

* The website builds an API with an OAuth 2.0 authorization server, a resource
  server, scoped access tokens, and a token lifecycle, and the agent uses the
  API. This is the correct architecture for a developer platform and an
  unrealistic ask for most websites, which have no API and no plans to build
  one.

This document takes a different position: the website already has the only
interface that matters, which is the website. What is missing is a standard
way for the user to authorize an agent to hold a session on it, and for the
website to know which agent is holding it.

The profile reuses two existing building blocks. The Device Authorization
Grant {{RFC8628}} is already the standard way for a client that cannot receive
a redirect to obtain user authorization: the client displays a short code, the
user approves on their own device, the client polls for the result. Client ID
Metadata Documents {{CIMD}} already remove the need for a developer portal: the
client identifier is a URL that the client's operator controls and the
authorization server fetches. Many websites already operate an OAuth 2.0
authorization server for their own first-party sign-in, with {{RFC8414}}
metadata, PKCE, refresh tokens, and a scoped consent screen. For such a
website, implementing this profile is a configuration change and one new HTTP
endpoint.

The one genuinely new element is the last step. The result of the device grant
is an access token. The agent needs a web session. This document defines a
`web_session_uri` to which the agent's browser presents the access token and
from which it receives the website's normal session cookie. The access token
is short-lived and serves only to bridge from the authorization server to the
session.


# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This document uses the terms "authorization server", "client", "access token",
"refresh token", and "scope" as defined in {{RFC6749}}, and "device
authorization endpoint", "device code", "user code", and "verification URI" as
defined in {{RFC8628}}.

Agent:
: Software that acts on behalf of a user by operating a web browser. In OAuth
  terms the agent is the client.

Agent Platform:
: The organization that operates the agent and publishes its Client ID
  Metadata Document.

Site:
: The website on which the agent needs a session. The site operates the
  authorization server and the `web_session_uri`.

User:
: The person on whose behalf the agent acts and who holds an account at the
  site.

Agent Session:
: A session at the site, established through this profile, that the site can
  attribute to both a user and an agent.


# Overview {#overview}

~~~
+-------+                                +------+      +--------+
| Agent |                                | Site |      |  User  |
+---+---+                                +--+---+      +---+----+
    |                                       |              |
    | (1) GET /.well-known/                 |              |
    |     oauth-authorization-server        |              |
    |-------------------------------------->|              |
    |<--------------------------------------|              |
    |                                       |              |
    | (2) POST device_authorization_endpoint|              |
    |     client_id=<CIMD URL>              |              |
    |     scope=web_session                 |              |
    |-------------------------------------->|              |
    |<-- device_code, user_code, ...  ------|              |
    |                                       |              |
    | (3) show verification_uri_complete and user_code     |
    |----------------------------------------------------->|
    |                                       |              |
    |                                       | (4) sign in, |
    |                                       |  review, and |
    |                                       |    approve   |
    |                                       |<-------------|
    |                                       |              |
    | (5) POST token_endpoint (poll)        |              |
    |     grant_type=...device_code         |              |
    |-------------------------------------->|              |
    |<-- access_token, web_session_uri -----|              |
    |                                       |              |
    | (6) GET web_session_uri               |              |
    |     Authorization: Bearer <token>     |              |
    |-------------------------------------->|              |
    |<-- Set-Cookie: <session> -------------|              |
    |                                       |              |
~~~
{: #fig-overview title="Agent Session Flow"}

1. The agent fetches the site's authorization server metadata and confirms
   that the site supports this profile ({{discovery}}).

2. The agent starts a Device Authorization Grant, identifying itself with a
   Client ID Metadata Document URL and requesting the `web_session` scope
   ({{authorization}}).

3. The agent shows the user the verification URI and user code.

4. On their own device, the user opens the verification URI, signs in to the
   site with whatever method the site already supports, reviews the agent's
   identity and the requested access, and approves.

5. The agent polls the token endpoint and receives an access token together
   with the `web_session_uri` ({{token-response}}).

6. The agent's browser presents the access token at the `web_session_uri` and
   receives the site's session cookie ({{exchange}}). The browser is now signed
   in as the user, and the site knows which agent holds the session.


# Discovery {#discovery}

A site supporting this profile MUST publish authorization server metadata
{{RFC8414}}. OpenID Connect Discovery {{OIDC-DISCOVERY}} metadata that contains
the same fields is equivalent.

An agent MUST treat a site as supporting this profile only if all of the
following hold in the metadata:

* `client_id_metadata_document_supported` is `true`.

* `device_authorization_endpoint` is present.

* `grant_types_supported` includes
  `urn:ietf:params:oauth:grant-type:device_code`.

* `token_endpoint_auth_methods_supported` includes `none`.

* `scopes_supported` includes `web_session`.

If `grant_types_supported` also includes `refresh_token`, the agent MAY expect
to mint additional sessions without further user interaction, as described in
{{refresh}}.

Agents MUST NOT hard-code endpoint locations for a site. All endpoints used by
this profile are obtained from the metadata document or from the token
response.


# Client Identification {#client}

The agent's `client_id` is a Client ID Metadata Document URL as defined in
{{CIMD}}. The document MUST include:

* `client_id`, equal to the document URL.

* `client_name`, the human-readable name the site displays on the consent
  screen.

* `client_uri`, a page describing the agent for users.

* `token_endpoint_auth_method` with the value `none`.

* `grant_types` including `urn:ietf:params:oauth:grant-type:device_code`.

The document SHOULD include `logo_uri` and `policy_uri`. Because this profile
uses no redirect, `redirect_uris` is not required and MAY be empty.

A single Agent Platform typically publishes one Client ID Metadata Document for
all of its agents. The site therefore knows the platform operating the agent,
not an individual agent instance. Finer attribution, when needed, is the
subject of {{request-identity}}.

The site MUST fetch and validate the Client ID Metadata Document as specified
in {{CIMD}}, including its restrictions on URL schemes, redirects, response
size, and caching. The site MUST NOT allow a Client ID Metadata Document client
to use any grant type other than the device code grant and the refresh token
grant unless the site has a separate policy permitting it.


# Authorization {#authorization}

The agent sends a device authorization request as defined in {{Section 3.1 of
RFC8628}} with `client_id` set to its Client ID Metadata Document URL and
`scope` containing `web_session`. The agent MAY request additional scopes that
the site defines, but the site MUST treat `web_session` as the scope that
authorizes {{exchange}}.

The device authorization response is as defined in {{Section 3.2 of RFC8628}}.
The site MUST include `verification_uri_complete`. Sites are encouraged to make
the verification URI their existing sign-in page with the user code carried as
a query parameter, so that the user signs in through the flow they already
know.

The agent displays `verification_uri_complete` and `user_code` to the user.
The user code MUST be displayed even when the complete URI is provided, so
that the user can confirm it against the code shown by the site.

## The web_session Scope {#scope}

The `web_session` scope requests authorization for the client to obtain, via
the `web_session_uri`, a session at the site with the same capabilities as a
session established by the user signing in directly, subject to the
restrictions in {{session-policy}}.

The consent screen MUST display, in language intended for the user:

* the `client_name` and the host of the `client_uri` from the Client ID
  Metadata Document;

* that approving will allow the agent to use the site as the user;

* how long the resulting session or sessions may last;

* how the user can revoke the authorization later.

The site MUST require the user to be authenticated before showing the consent
screen and MUST require an explicit approval action. A user code carried in
`verification_uri_complete` MUST NOT be treated as approval.

## Polling {#polling}

The agent polls the token endpoint as defined in {{Section 3.4 of RFC8628}},
using `grant_type=urn:ietf:params:oauth:grant-type:device_code`, and MUST
honour the `authorization_pending` and `slow_down` responses defined in
{{Section 3.5 of RFC8628}}. Because the client authenticates with the method
`none`, the token request carries the `client_id` parameter.


# Token Response {#token-response}

On approval, the site returns a token response as defined in {{Section 5.1 of
RFC6749}} with the following requirements:

`access_token`:
: REQUIRED. A bearer token {{RFC6750}} whose only defined use is presentation at
  the `web_session_uri`.

`token_type`:
: REQUIRED. The value `Bearer`.

`expires_in`:
: REQUIRED. The lifetime of the access token in seconds. The value MUST NOT
  exceed 300.

`scope`:
: REQUIRED if different from the requested scope, as in {{RFC6749}}. MUST
  include `web_session` if the exchange is authorized.

`web_session_uri`:
: REQUIRED when `scope` includes `web_session`. An HTTPS URL at which the
  access token can be exchanged for a session, as described in {{exchange}}.
  The URL MUST be on an origin that the site intends to set its session cookie
  for.

`refresh_token`:
: OPTIONAL. See {{refresh}}.

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "web_session",
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "web_session_uri": "https://www.example.com/agent-session"
}
~~~
{: #fig-token title="Token Response"}

`web_session_uri` is carried in the token response rather than in the
authorization server metadata so that the site can vary it per client, per
user, or per deployment without changing a published document, and so that the
value is only available to a client that has completed authorization.


# Session Exchange {#exchange}

The agent's browser presents the access token at the `web_session_uri`. The
site MUST support both of the following request forms. The access token MUST
NOT be placed in the request URL.

~~~
GET /agent-session HTTP/1.1
Host: www.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
~~~
{: #fig-get title="Exchange using the Authorization header"}

~~~
POST /agent-session HTTP/1.1
Host: www.example.com
Content-Type: application/x-www-form-urlencoded

access_token=2YotnFZFEjr1zCsicMWpAA
~~~
{: #fig-post title="Exchange using a form body"}

The header form suits browsers driven through an automation protocol, where
the agent can attach a header to a navigation. The form body suits agents that
can only cause the browser to submit a form.

On success, the site MUST set its session cookie or cookies with `Set-Cookie`
{{RFC6265}} and respond with either a `200 (OK)` response or a `302 (Found)`
redirect {{RFC9110}} to a same-origin location. The site MAY support an optional `next`
query parameter naming a same-origin path to redirect to; the site MUST reject
any `next` value that is not a same-origin path.

The response MUST include `Cache-Control: no-store`.

If the access token is missing, expired, not issued for the `web_session`
scope, or otherwise invalid, the site MUST respond with `401 (Unauthorized)`
and a `WWW-Authenticate: Bearer` header as defined in {{RFC6750}}, with the
`error` value `invalid_token`.

The `web_session_uri` MUST NOT be protected by bot detection, CAPTCHA, or
browser challenge mechanisms. The access token is the proof that the request
is authorized; a challenge at this endpoint would defeat the single step that
distinguishes an approved agent from unidentified automation. The site MAY
apply rate limits to the endpoint.

The site is not required to make the access token single-use. The `expires_in`
bound in {{token-response}} limits the exposure window, and a token that has
been presented once has already served its purpose.


# Agent Session Properties {#session}

## Attribution {#attribution}

A session created through {{exchange}} MUST be associated, in the site's
session store or equivalent, with both the user who approved the grant and the
`client_id` of the agent. This association is what allows the site to apply
agent-specific policy, risk scoring, and audit.

## Policy {#session-policy}

An Agent Session is, by default, a normal session at the site. The site MAY
restrict actions available to Agent Sessions. Sites are encouraged to adopt a
small fixed policy rather than a scope vocabulary; a recommended default
denies the following from an Agent Session:

* changing the account's password, email address, phone number, or
  authentication factors;

* adding payment instruments;

* closing or deleting the account;

* approving further agent authorizations.

## Lifetime {#lifetime}

The lifetime of an Agent Session is site policy and MUST be disclosed on the
consent screen. It need not match the lifetime of sessions established by
direct sign-in.

If the site lists active sessions to the user, Agent Sessions MUST appear in
that list, identified as such, with the agent's `client_name`.

## Refresh {#refresh}

A site MAY issue a refresh token in the token response. A refresh token allows
the agent to obtain a new `web_session` access token, and therefore a new
session, without further user interaction, for example to sign in a new
browser instance after the previous one was discarded.

The lifetime of the refresh token bounds the delegation as a whole and MUST be
disclosed on the consent screen as the period during which the agent may sign
in without asking again. Refresh tokens issued under this profile MUST be bound
to the `client_id` that obtained them.

## Revocation {#revocation}

Revoking the delegation SHOULD be possible both through the site's user
interface for connected applications and, where the site implements it,
through token revocation {{RFC7009}} of the refresh token.

Revoking the delegation SHOULD terminate Agent Sessions created from it.
Terminating an individual Agent Session, for example from a list of active
sessions, need not revoke the delegation; the agent can mint another session
if it still holds a valid refresh token, and the user who wants to stop the
agent entirely revokes the delegation.

## Identifying the Agent per Request {#request-identity}

This profile identifies the agent at the moment a session is established. A
site that wants to identify the agent on every request, or to distinguish
agent instances operated by the same platform, can require that requests made
within an Agent Session carry HTTP message signatures as described in {{WBA}}.
Such a requirement is out of scope for this document.


# Relationship to Other Approaches {#relationship}

Authorization Code Grant:
: The authorization code grant {{RFC6749}} requires the client to receive a
  redirect. Agents often run in environments with no stable, reachable
  redirect URI, and the redirect would in any case arrive at the user's
  device rather than the agent's. The device grant avoids both problems and
  places the approval on the site's own origin.

Dynamic Client Registration:
: Dynamic Client Registration {{RFC7591}} also removes the need for a developer
  portal, but it requires the site to operate a registration endpoint, store
  per-client state, and often to issue client secrets. Client ID Metadata
  Documents move that state to the client's operator.

API Access Tokens:
: This profile deliberately does not define how the `web_session` access token
  could be used against an API. A site that later exposes an API can add
  scopes and a resource server to the same authorization server without
  changing this profile.

Native Applications:
: {{RFC8252}} addresses public clients that can open a browser and receive a
  redirect on the same device as the user. Agents typically operate a browser
  that is not on the user's device, which is why the device grant is used
  instead.


# Security Considerations {#security}

## Consent Phishing {#consent-phishing}

An agent, or an attacker posing as one, can send a user a verification URI in
the hope that the user approves without reading. This is the central risk of
the device grant and is discussed in {{Section 5.4 of RFC8628}}. The
mitigations belong to the site: display the `client_name` and `client_uri`
host prominently, display the user code and require the user to confirm it,
expire user codes within a short period (ten minutes is RECOMMENDED), and rate
limit device authorization requests per `client_id` and per source address.
Because the user signs in and approves on the site's own origin, the site has
full control of this screen.

## Client Identity {#client-identity}

A Client ID Metadata Document proves control of a URL, not the trustworthiness
of the party controlling it. Sites SHOULD display the host of the `client_id`
on the consent screen and MAY maintain allow lists or deny lists of client
identifier hosts. Sites fetching Client ID Metadata Documents MUST follow the
server-side request forgery protections in {{CIMD}}.

## Access Token Exposure {#token-exposure}

The `web_session` access token is equivalent, for its lifetime, to the ability
to sign in as the user. Its exposure is bounded by three properties: the
lifetime limit of 300 seconds in {{token-response}}; the prohibition on
carrying it in a URL in {{exchange}}, which keeps it out of browser history,
server logs, and `Referer` headers; and the `device_code` binding in
{{RFC8628}}, which ensures that only the agent that started the flow can obtain
it in the first place. A leaked user code or verification URI yields nothing
without the corresponding device code.

## Login Cross-Site Request Forgery {#login-csrf}

An attacker who can cause a victim's browser to make a request to the
`web_session_uri` with the attacker's access token could sign the victim's
browser into the attacker's account. The header form in {{exchange}} is not
reachable by a cross-site navigation because browsers do not attach custom
headers to navigations. Sites supporting the form body MUST either require a
same-origin `Origin` or `Sec-Fetch-Site` header, or require that the request
be made with a method and content type that cross-site HTML forms cannot
produce. Sites MUST NOT accept the access token in a URL query parameter.

## Session Exfiltration {#exfiltration}

Once established, an Agent Session is a cookie held by a browser the agent
controls. If that browser is compromised, the cookie can be exfiltrated like
any other session cookie. This profile does not prevent that; it ensures the
site can attribute the session to an agent and apply {{session-policy}}, and it
gives the user a revocation path that does not involve changing their
password. Sites wanting per-request assurance should consider
{{request-identity}}.

## Bot Detection {#bot-detection}

The requirement in {{exchange}} that the `web_session_uri` not be protected by
bot detection applies only to that endpoint. The site remains free to apply
any detection it wishes to the rest of an Agent Session, and the attribution in
{{attribution}} gives it a basis for treating identified agents differently
from unidentified automation.

## General OAuth Security {#general-security}

The recommendations of {{RFC9700}} apply to authorization servers implementing
this profile.


# Privacy Considerations {#privacy}

The site learns which Agent Platform a user has delegated to, and the Agent
Platform learns which sites a user has delegated access to. Neither learns
more than it would if the user had shared their password, and the user gains
the ability to see and revoke the delegation at the site. Sites SHOULD retain
the association between sessions and `client_id` only as long as needed for
security and audit.


# IANA Considerations {#iana}

## OAuth Parameters Registration {#iana-parameter}

This document requests registration of the following value in the "OAuth
Parameters" registry established by {{RFC6749}}.

Parameter name:
: `web_session_uri`

Parameter usage location:
: token response

Change controller:
: IETF

Reference:
: This document

Scope values are not registered with IANA. The `web_session` scope is defined
in {{scope}}.


# Open Issues {#open-issues}

* Whether `web_session_uri` should additionally be advertised in authorization
  server metadata so that an agent can confirm support before involving the
  user.

* Whether the profile should define a way for the agent to request a session
  lifetime, or whether lifetime should remain purely site policy.

* Whether the recommended default policy in {{session-policy}} should be
  normative.

* Whether to define a standard way for the site to signal, to the agent's
  browser, that a session has been revoked, beyond the cookie ceasing to work.


--- back

# Example: Site with an Existing Authorization Server {#example}
{:numbered="false"}

Many consumer websites already operate an OAuth 2.0 authorization server for
their own sign-in, with the website itself as a confidential first-party
client. Such a server typically publishes {{RFC8414}} metadata with an
authorization endpoint, a token endpoint, PKCE, refresh tokens, revocation, and
a scoped consent screen, and lacks only the following relative to this
profile:

* `client_id_metadata_document_supported: true` and the corresponding client
  lookup;

* a device authorization endpoint and the device code grant;

* `none` in `token_endpoint_auth_methods_supported` for Client ID Metadata
  Document clients;

* the `web_session` scope and its consent text;

* the `web_session_uri` endpoint, which validates a bearer token and performs
  the same session establishment the site already performs after a first-party
  sign-in.

For such a site, the first four items are configuration of an existing
component and the fifth is a single new route.


# Acknowledgments
{:numbered="false"}

This profile is assembled almost entirely from the work of the authors of
{{RFC8628}} and {{CIMD}}.
