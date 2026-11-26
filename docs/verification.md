# Verification

Run on the apps host unless noted.

## The stack is healthy

```bash
sudo docker ps --filter name=authentik --format '{{.Names}}\t{{.Status}}'
# authentik-server, authentik-worker and authentik-postgres are up; postgres is healthy
```

## The environment is complete

```bash
sudo docker exec authentik-server env | grep -c '^AUTHENTIK_'
# Compare against the names in files/authentik.env.example
sudo docker exec authentik-server env | grep AUTHENTIK_ERROR_REPORTING__ENABLED
# AUTHENTIK_ERROR_REPORTING__ENABLED=false
```

## Only the allowed networks can reach it

```bash
# From an allowed network
curl -s -o /dev/null -w '%{http_code}\n' https://auth.example.net/
# 200 (or 302 to the sign-in flow)

# From a network that is not on the list
curl -s -o /dev/null -w '%{http_code}\n' https://auth.example.net/
# 403
```

## Password reset by email

Sign out, choose "Forgot password?", enter a username and open the link from the message. The link should arrive within a minute, work once, and a second use should fail.

## Recovery link

```bash
sudo docker exec authentik-server ak create_recovery_key 10 admin
# prints /recovery/use-token/...  open it, confirm you are signed in, then open it again and confirm it no longer works
```

## Accounts and groups

Read-only listing, from the shell:

```bash
sudo docker exec authentik-server ak shell -c \
  "from authentik.core.models import User; [print(u.username, u.is_active, u.is_superuser) for u in User.objects.all()]"
# the built-in admin account is False; exactly one named superuser is True
```

## No leftover errors in the worker

```bash
sudo docker logs authentik-worker --since 10m 2>&1 | grep -c DockerException
# 0. The worker has no Docker socket, so no outpost integration should point at one.
```

## Forward-auth

From an allowed management network, open the protected code editor in a private window. It should send you to the sign-in page, and after signing in, back to the editor. The same session should open the Traefik dashboard without a second prompt.

## SAML and local fallback

Open the SIEM dashboard and confirm it redirects to Authentik and returns signed in. Open its local login address and confirm that still works.
