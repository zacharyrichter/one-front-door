# Design

## Shape

One compose stack on the apps host: the Authentik server, the worker and a dedicated PostgreSQL container. The server joins two networks, a private one shared with the database and the proxy network that Traefik uses. Traefik routes `auth.example.net` to it. All configuration comes from an environment file next to the compose file.

The stack is deployed by hand, not by the automated stack syncer. It is the thing every other login depends on, so a change to it is a deliberate act.

## Internal-only access

The router for `auth.example.net` carries an IP allow-list middleware. It permits:

| Source | Why |
|---|---|
| Loopback and the Docker networks | The proxy itself and containers on the host |
| Management, apps and user networks | The people and services that sign in |
| WireGuard client range | Remote access comes in over the VPN, never directly |

Anything else gets a 403 from Traefik. The same Traefik instance requires TLS 1.3 and strict SNI, so a client has to ask for a known name.

The Git server's route has one extra allowed address: the DMZ host that pulls stacks. That is a separate middleware, so the allow-list on Authentik stays as narrow as above.

## Identity model

| Group | Purpose |
|---|---|
| `admins` | Superuser group. Inherits from the built-in `authentik Admins` |
| `users` | Everyone in the household |
| `gitadmin` | Admin rights in the Git server |
| `wazuh-administrators` | Administrator role in the SIEM. Inherits from `admins` |

- One person is a superuser; the three other household members are ordinary users.
- The built-in `akadmin` account is disabled. Nothing depends on it.
- Two service accounts belong to the proxy outpost and are managed by Authentik.

## Sign-in flow

The default authentication flow is four stages:

1. **Identification:** email or username, with "remember me".
2. **Password:** five failed attempts and the flow ends.
3. **Authenticator validation:** TOTP, WebAuthn, static codes and the other classes are accepted, and a user with none enrolled is let through.
4. **Login:** sessions last 30 days.

The third stage is why MFA is optional. It is a policy decision, not a gap: the stack serves internal applications only, most of the people using it would be deterred by a mandatory second factor, and the allow-list already keeps the page off the internet. If the stack were public, the "not configured" action would change from skip to force enrolment, and that is the only change needed.

## Three ways an app signs in

| Pattern | Used for | How it works |
|---|---|---|
| **OIDC** (16 apps) | Apps with a "log in with OpenID" option | The app redirects the browser to Authentik, gets a code, and exchanges it for tokens. |
| **SAML** (1 app) | The SIEM dashboard | Authentik signs an assertion for the dashboard's audience. A local login remains as a fallback. |
| **Forward-auth proxy** (1 app, several routes) | Management web tools with no login of their own | Traefik asks the outpost whether the request is signed in before passing it on. |

### OIDC settings

- **Confidential clients.** Each app has its own client ID and secret.
- **Issuer per provider.** Each app sees its own issuer address, so a discovery document cannot be reused by another app.
- **Subject mode.** The subject is a hashed user ID, so the identifier an app stores reveals nothing about the account. One app that matches users by email address uses the email instead.
- **Exact redirect addresses.** No wildcards.
- **Lifetimes.** Access tokens five minutes, authorization codes one minute, refresh tokens 30 days.
- **Signing key.** A self-signed certificate pair that Authentik generates, used only to sign tokens for its own clients.
- **Scopes.** The default openid, email, profile and offline-access scopes, plus small custom mappings where an app expects a particular claim.

### Forward-auth in front of the management tools

A proxy outpost container runs on the management host. The management Traefik instance has a `forwardAuth` middleware that sends each request to the outpost and copies the username, groups, email, name and ID headers from its answer. The cookie domain is the base domain, so one sign-in covers every protected route.

It protects the browser-based code editor, the remote browser used to reach other management interfaces, and the Traefik dashboard. The remote browser has a second layer in front of it: an allow-list of four source addresses that is checked before the login is.

## Who can open what

Each application has policy bindings in "any" mode, so being in any listed group is enough. Most applications are bound to `admins` and `users`. The exceptions are narrower: the Git server to `admins` and `gitadmin`, the container manager to `admins` only, the SIEM to `wazuh-administrators`, and the management proxy to a single named user, and the monitoring dashboard to that user plus `admins`. One application holding sensitive personal data is bound to a single user.

## No alerts, on purpose

The three default notification rules have no recipient, so Authentik sends nothing. Mail is configured for exactly one job, password reset. The update alert that once existed was retired along with its notification token.

## Backups

The apps VM is backed up weekly in stop mode, so the database files are copied at rest. The backup covers the Authentik database, data, certificate and template directories. The recovery flow and every provider live in that database.
