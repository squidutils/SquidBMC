# Authentication

SquidBMC has three ways to sign in, which can be used in any combination:

- OpenID Connect (OIDC) single sign-on with any identity provider
- GitHub OAuth
- A local username and password

The login page shows a button for each method that is configured: OIDC first,
then GitHub, then the local form.

## When login is enforced

Login is enforced as soon as at least one method is configured, meaning any of
`OIDC_ISSUER`, `GITHUB_CLIENT_ID` or `PLATFORM_USER` is set.

If none is set the app runs **open**: anyone who can load the page can use the
consoles and power controls. The login page then says "No auth configured".
This is intended only for a first start on a trusted workstation. Always set up
a login before exposing the app anywhere.

Once login is enforced, every route except `/login`, `/logout` and `/auth/*`
redirects to `/login` until you sign in. The two WebSocket endpoints, the KVM
stream (`/websockify`) and the text console (`/solws`), return `401` without a
signed-in session. The static noVNC and xterm.js files are served without a
login; they are public code and do nothing without an authenticated WebSocket.

## Local login

```
PLATFORM_USER=admin
PLATFORM_PASS=<a long random password>
```

- Both values are compared in constant time (`Rack::Utils.secure_compare`).
- A wrong username or password shows "Invalid credentials".
- Useful when the host has no internet access, for example on an isolated
  management VLAN. GitHub login needs both the browser and the server to reach
  github.com; OIDC needs both to reach your identity provider.
- Leave `PLATFORM_USER` empty to turn local login off.

## GitHub OAuth

1. On GitHub go to **Settings -> Developer settings -> OAuth Apps** and register
   a new OAuth app.
2. Set the authorization callback URL to `<base URL>/auth/github/callback`, for
   example `http://localhost:8088/auth/github/callback` locally or
   `https://bmc.example.com/auth/github/callback` behind a proxy. One callback
   URL per app, so use the exact host and port you browse from.
3. Put the app's credentials and your allow-list in `.env`:

```
GITHUB_CLIENT_ID=<client id>
GITHUB_CLIENT_SECRET=<client secret>
GITHUB_ALLOWED_USERS=your-github-login,another-login
```

- The app requests the `read:user` scope only.
- `GITHUB_ALLOWED_USERS` is a comma-separated, case-insensitive list of GitHub
  logins. A valid GitHub user who is not on the list is rejected with `403` and
  "<login> is not authorised".
- Behind an HTTPS proxy, set `TRUST_PROXY=1`, otherwise the callback URL the
  app sends to GitHub starts with `http://` and does not match the registered
  one (see [Reverse proxy and TLS](reverse-proxy.md)).

## OpenID Connect (any identity provider)

Works with any standards-compliant provider, for example Authentik, Keycloak,
Microsoft Entra ID or Google.

1. At your identity provider, register a **confidential** client (one with a
   client secret).
2. Register the redirect URI `https://<your-host>/auth/oidc/callback`. Providers
   match it exactly, including scheme, host, port and path.
3. Make sure the provider includes a `groups` claim if you want to allow
   groups (see below).
4. Configure `.env`:

```
OIDC_ISSUER=https://idp.example.com/application/o/squidbmc/
OIDC_CLIENT_ID=<client id>
OIDC_CLIENT_SECRET=<client secret>
OIDC_REDIRECT_URI=https://bmc.example.com/auth/oidc/callback
OIDC_ALLOWED_USERS=alice,ops@example.com
OIDC_ALLOWED_GROUPS=BMC Admins
OIDC_LABEL=Example SSO
```

How it works:

- The provider's settings are found by discovery, from
  `<OIDC_ISSUER>/.well-known/openid-configuration`. Use the issuer URL exactly
  as your provider publishes it.
- The authorisation code flow with PKCE is used. The scopes requested are
  `openid`, `email` and `profile`.
- Starting a sign-in is a CSRF-protected `POST` to `/auth/oidc` (the same for
  GitHub at `/auth/github`). Plain `GET` requests to start a sign-in are not
  accepted.
- `OIDC_REDIRECT_URI` is sent to the provider as-is, so it does not depend on
  `TRUST_PROXY`. It must still be the address your browser actually uses.
- On a successful sign-in the session is reset before the user is logged in,
  so an old session cannot be reused (no session fixation).
- If OIDC is not configured, `/auth/oidc/callback` returns `404`.

### Allow-lists (fail-closed)

A user who signs in successfully at the provider is let in only if one of these
is true:

- Their nickname, `preferred_username` or `email` is in `OIDC_ALLOWED_USERS`
  (comma-separated, case-insensitive).
- One of the values in their `groups` claim is in `OIDC_ALLOWED_GROUPS`
  (comma-separated). Group names must match exactly, including case. A partial
  match does not count.

Anyone else is rejected with `403` and "<name> is not authorised". If both
lists are empty, nobody can sign in through OIDC. This is deliberate
("fail-closed"): a misconfiguration locks people out rather than letting
everyone with an account at your provider in.

## Sessions

- Sessions are stored in a signed cookie (`Rack::Session::Cookie`), signed with
  `SESSION_SECRET`. Generate it with `openssl rand -hex 32`.
- If `SESSION_SECRET` is not set, a random secret is generated at each start.
  That is safe, but everyone is logged out whenever the container restarts.
- The cookie is `SameSite=Lax` and expires after 12 hours.
- The cookie is marked `Secure` only when `TRUST_PROXY=1`. Do not set
  `TRUST_PROXY=1` while browsing over plain HTTP: the browser would not send the
  cookie back and login would silently fail.
- **Log out** clears the session.

There is no separate CSRF token on the app's forms; the `SameSite=Lax` cookie
is the mitigation. See [SECURITY.md](../SECURITY.md#csrf-posture).

## Sign-in error messages

- "Invalid credentials" - wrong local username or password.
- "<name> is not authorised" - signed in at GitHub or the OIDC provider, but not
  on the allow-list.
- "GitHub sign-in failed" or "Sign-in failed" - the provider returned no
  usable identity.
- "GitHub sign-in failed: <message>" or "Single sign-on failed: <message>" -
  the provider or the OAuth library reported an error; the message says which.

See [Troubleshooting](troubleshooting.md#login-and-single-sign-on) for causes
and fixes.
