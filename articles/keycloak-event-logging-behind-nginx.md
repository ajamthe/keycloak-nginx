# Tracking Real Client IPs in Keycloak Login Events — Even Behind a Proxy

Keycloak has built-in event logging, but getting it to work correctly in a production-like setup — where Keycloak sits behind a reverse proxy — requires a few deliberate steps. Without them, every login event logs the proxy's IP instead of the real client IP, making your audit trail useless.

This article walks through enabling Keycloak event logging and ensuring the IP address captured is always the real one.

---

## The Problem

By default, Keycloak logs no login or logout events to its server log. And even if you turn on event logging, running Keycloak behind nginx (or any reverse proxy) means every event shows the proxy container's IP — not your user's IP. That's a security blind spot.

---

## Step 1: Enable Event Logging in Keycloak

Keycloak ships with the `jboss-logging` event listener, but it's not wired up out of the box.

In the **Admin Console**, go to:
**Realm Settings → Events → Event listeners** and add `jboss-logging`.

Then under **Event types**, enable at minimum: `LOGIN`, `LOGIN_ERROR`, `LOGOUT`.

This alone is not enough. By default, the `org.keycloak.events` log category is suppressed. You need to explicitly enable it at the server level:

```
KC_LOG_LEVEL=INFO,org.keycloak.events:DEBUG
```

---

## Step 2: Put Keycloak Behind nginx and Fix the IP

The nginx config is straightforward — the critical part is forwarding the real client IP:

```nginx
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Real-IP         $remote_addr;
```

On the Keycloak side, two environment variables are needed:

```bash
KC_PROXY_HEADERS=xforwarded   # Trust X-Forwarded-* headers from the proxy
KC_HOSTNAME=http://localhost  # The public-facing URL (nginx's address)
KC_HTTP_ENABLED=true          # Keycloak speaks plain HTTP; nginx handles TLS
```

Keycloak's port should **not** be published directly to the host — all traffic must flow through nginx.

The full `docker-compose.yml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - keycloak
    networks:
      - keycloak-net

  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
      KC_HOSTNAME: http://localhost
      KC_HTTP_ENABLED: true
      KC_PROXY_HEADERS: xforwarded
      KC_LOG_LEVEL: "INFO,org.keycloak.events:DEBUG"
    expose:
      - "8080"       # internal only — not published to host
    command: start-dev
    networks:
      - keycloak-net

networks:
  keycloak-net:
    driver: bridge
```

---

## Step 3: Verify It Works

Here is what you actually see in the logs after a successful login, a token exchange, and a failed login attempt:

**Successful login:**

```
2026-03-19 12:18:52,148 DEBUG [org.keycloak.events] (executor-thread-3)
type="LOGIN", realmName="master", clientId="security-admin-console",
userId="d2d39d35-ff06-48bb-a58a-f7a8689f791c",
sessionId="ucaINE0oBE34KyruvbsQ0PdR",
ipAddress="172.18.0.1",
username="admin", auth_method="openid-connect",
redirect_uri="http://localhost/admin/master/console/"
```

**Authorization code exchanged for token:**

```
2026-03-19 12:18:52,329 DEBUG [org.keycloak.events] (executor-thread-4)
type="CODE_TO_TOKEN", realmName="master", clientId="security-admin-console",
userId="d2d39d35-ff06-48bb-a58a-f7a8689f791c",
sessionId="ucaINE0oBE34KyruvbsQ0PdR",
ipAddress="172.18.0.1",
grant_type="authorization_code", scope="openid email profile"
```

**Failed login attempt:**

```
2026-03-19 12:31:29,144 WARN  [org.keycloak.events] (executor-thread-1)
type="LOGIN_ERROR", realmName="master", clientId="security-admin-console",
ipAddress="172.18.0.1",
error="not_allowed", reason="Client not allowed for direct access grants"
```

The key field is `ipAddress`. It reads `172.18.0.1` — the Docker host (the real client) — not `172.18.0.3`, which is the nginx container's IP. That distinction is the whole point.

---

## How to Confirm the Proxy Is Being Trusted

A quick sanity check — inspect the nginx container's IP on the internal network:

```bash
docker inspect keycloak-nginx \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# → 172.18.0.3
```

If `ipAddress` in the logs matches the nginx container IP, Keycloak is ignoring `X-Forwarded-For` and logging the proxy's IP. If it shows a different address — the gateway or a real client IP — Keycloak is correctly reading the forwarded header.

---

## Summary

| What | How |
|---|---|
| Enable event logging | Add `jboss-logging` listener in Admin Console |
| Surface events in server log | `KC_LOG_LEVEL=INFO,org.keycloak.events:DEBUG` |
| Forward real IP via nginx | `X-Forwarded-For`, `X-Real-IP` headers in nginx config |
| Tell Keycloak to trust the proxy | `KC_PROXY_HEADERS=xforwarded` |

Three environment variables and one nginx config block — that is all it takes to get accurate, trustworthy login audit logs from Keycloak in a proxied deployment.
