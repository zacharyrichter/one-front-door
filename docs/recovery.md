# Recovery

Two situations, two paths.

## Forgotten password

Any user opens the sign-in page, chooses "Forgot password?", and enters their username. Authentik emails a single-use link that expires after 30 minutes. The link leads to a page where they set a new password and are signed in.

Details of the flow:

- It is identified by username only.
- It is `require_unauthenticated`, so a signed-in session cannot start it.
- A reset **does not reactivate a disabled account**. A disabled account stays disabled, so a reset link cannot undo a deliberate lock.
- Mail goes through an SMTP relay on implicit TLS (port 465). The stack's mail settings are in [`files/authentik.env.example`](../files/authentik.env.example).

The flow is imported once through **Flows → Import** using [`files/recovery-flow.yaml`](../files/recovery-flow.yaml), then selected as the "Recovery flow" on the identification stage. After that it belongs to the database like any other flow, and edits in the web interface stay.

## Admin account locked out or disabled

This is the case where email will not help, because the account that would receive the reset is the one that is disabled, or the provider is unusable.

There is no standing admin password. The recovery path is a shell on the apps host, which needs an SSH certificate from the bastion:

```bash
# 1. Make sure the account is active (prints 1 when it changed one row)
sudo docker exec authentik-server ak shell -c \
  "from authentik.core.models import User; print(User.objects.filter(username='admin').update(is_active=True))"

# 2. Create a single-use recovery link, valid for 10 minutes
sudo docker exec authentik-server ak create_recovery_key 10 admin
```

The second command prints a path beginning `/recovery/use-token/`. Put `https://auth.example.net` in front of it and open it from an allowed network. The link signs in once.

- **The duration is in minutes.**
- **The link works once.**
- **The built-in `akadmin` account stays disabled.** Recovery reactivates a named account, and only for as long as it takes to get back in.

## What it does not help with

- A lost SSH certificate. That is a separate break-glass path, because the shell is the thing this one needs.
- A stopped database or an unreachable host. Those are restored from backup.
- Apps with their own local login (the SIEM and the home automation system) keep that login as a fallback and do not need Authentik to get in.
