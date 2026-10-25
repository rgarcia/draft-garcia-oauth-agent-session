<!-- regenerate: off -->

# OAuth 2.0 Agent Session

This is the working area for the individual Internet-Draft, "OAuth 2.0 Agent Session".

* [Editor's Copy](https://rgarcia.github.io/draft-garcia-oauth-agent-session/#go.draft-garcia-oauth-agent-session.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-garcia-oauth-agent-session)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-garcia-oauth-agent-session)
* [Compare Editor's Copy to Individual Draft](https://rgarcia.github.io/draft-garcia-oauth-agent-session/#go.draft-garcia-oauth-agent-session.diff)


## Summary

Agent Session lets a software agent obtain an ordinary logged-in web session on
a website on behalf of a user, with the user's consent and without the user
sharing credentials. It is a profile of the OAuth 2.0 Device Authorization
Grant (RFC 8628) and OAuth Client ID Metadata Documents, and adds two things:
the `web_session` scope and the `web_session_uri` token response parameter.
A website that already runs an OAuth 2.0 authorization server for its own
sign-in implements it with configuration plus one new endpoint. No API, no
resource server.


## Companion Material

* [Decision log](decision-log.md) : design decisions and the alternatives considered
* [Existing authorization servers](notes/existing-authorization-servers.md) : a worked example of a consumer site that is one endpoint away from the profile


## Contributing

See the
[guidelines for contributions](https://github.com/rgarcia/draft-garcia-oauth-agent-session/blob/main/CONTRIBUTING.md).

The contributing file also has tips on how to make contributions, if you
don't already know how to do that.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

