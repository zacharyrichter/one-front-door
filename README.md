# one-front-door

**Every internal web app signs in through one identity provider, and the provider itself is reachable only from inside the network. Recovery from a locked-out admin account is a tested, two-command procedure that needs SSH access, not a standing password.**

Authentik runs as a three-container stack on the apps host. Eighteen applications sit behind it: sixteen over OpenID Connect, one over SAML, and one forward-auth proxy that puts a login in front of the management web tools. Access is decided by group membership, and the sign-in page is only reachable from the internal networks.

```
browser ─▶ Traefik (allow-list: internal networks only)
              │
              ▼
          Authentik ──▶ OIDC apps (16)      app redirects here to sign in
                    ├─▶ SAML app (1)        signed assertions
                    └─▶ proxy outpost (1)   forward-auth in front of the management web tools
```

## The ideas that matter

1. **The identity provider is internal-only, and the proxy enforces it.** The router in front of Authentik carries an IP allow-list. A request from anywhere else gets a 403 before Authentik sees it.
2. **Access is a group, not a per-app account.** Each application is bound to one or two groups, and the groups are the list to change when someone joins or leaves.
3. **Each application uses the lightest protocol it supports.** OIDC where the app speaks it, SAML where only that works, forward-auth where the app has no login of its own.
4. **Tokens are short and clients are confidential.** Access tokens last five minutes, authorization codes one minute, and every redirect address is an exact match.
5. **MFA is a deliberate choice for this scope.** The stack serves internal apps only, and the household would be deterred by it, so enrolment is optional. If the stack were reachable from the internet, it would be required.
6. **There is no standing admin password.** The built-in admin account is disabled. Recovery goes through a shell on the host, which already requires an SSH certificate.
7. **The provider sends no alerts by design.** Mail is configured for password resets only.

## What's here

| Path | What it is |
|---|---|
| [`docs/design.md`](docs/design.md) | The identity model, the three integration patterns, token settings and the decisions behind them |
| [`docs/recovery.md`](docs/recovery.md) | Password reset by email and recovering a locked-out admin |
| [`docs/verification.md`](docs/verification.md) | The checks that show each part is working |
| [`files/recovery-flow.yaml`](files/recovery-flow.yaml) | The password-recovery flow, imported once through the web interface |
| [`files/authentik.env.example`](files/authentik.env.example) | The environment variable names the stack reads, with placeholder values |

## Limits

- **The recovery flow is imported, not managed.** It is applied once and then lives in Authentik's database like the rest of the configuration, so it is backed up with the database rather than rebuilt from the repo.
- **Forward-auth covers three tools.** The management applications that have their own login (the monitoring dashboard, the Git server's admin pages and the container manager) use OIDC or a local login instead.
- **Internal-only is a network control.** Anyone already inside the allowed networks can reach the sign-in page. Group membership and the password are what stop them from there.

## Related

- [homelab-compose](https://github.com/zacharyrichter/homelab-compose): the compose files for the Authentik stack and the proxy outpost.
- [air-gap-the-keys](https://github.com/zacharyrichter/air-gap-the-keys): the management network the forward-auth proxy protects.

Everything here is sanitized: the domain is `example.net`, addresses are `10.0.x.x`, the human account is `admin`, and secrets are placeholders.

MIT licensed. See [`LICENSE`](LICENSE).
