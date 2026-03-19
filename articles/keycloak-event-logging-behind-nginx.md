# Tracking Real Client IPs in Keycloak Login Events — Even Behind a Proxy

Keycloak has built-in event logging, but getting it to work correctly in a production-like setup — where Keycloak sits behind a reverse proxy — requires a few deliberate steps. Without them, every login event logs the proxy's IP instead of the real client IP, making your audit trail useless.

This article walks through enabling Keycloak event logging, ensuring the IP address captured is always the real one, and handling a PII concern that only surfaces once you look at real logs.

---

## The Problem

By default, Keycloak logs no login or logout events to its server log. And even if you turn on event logging, running Keycloak behind nginx (or any reverse proxy) means every event shows the proxy container's IP — not your user's IP. That's a security blind spot.

---

## Step 1: Enable Event Logging in Keycloak

Keycloak ships with the `jboss-logging` event listener, but it's not wired up out of the box.

In the **Admin Console**, go to:
**Realm Settings → Events → Event listeners** and add `jboss-logging`.

Then under **Event types**, enable at minimum: `LOGIN`, `LOGIN_ERROR`, `LOGOUT`.

This alone is not enough. By default, the `org.keycloak.events` log category is suppressed at `INFO` level, and Keycloak emits login events at `DEBUG`. You need to explicitly enable it at the server level:

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

Here is what you actually see in the logs across a full session — login, token exchange, and logout:

**Successful login (with email):**

```
2026-03-19 21:56:04,047 DEBUG [org.keycloak.events] (executor-thread-11)
type="LOGIN", realmName="master", clientId="security-admin-console",
userId="7ab6219a-9145-46c4-9a9f-e63618c7b5e5",
sessionId="wUNuKbNoAA8RY9uIuAIYNxBC",
ipAddress="172.18.0.1",
username="ajamthe@gmail.com", auth_method="openid-connect",
redirect_uri="http://localhost/admin/master/console/"
```

**Authorization code exchanged for token:**

```
2026-03-19 21:56:04,250 DEBUG [org.keycloak.events] (executor-thread-14)
type="CODE_TO_TOKEN", realmName="master", clientId="security-admin-console",
userId="7ab6219a-9145-46c4-9a9f-e63618c7b5e5",
sessionId="wUNuKbNoAA8RY9uIuAIYNxBC",
ipAddress="172.18.0.1",
grant_type="authorization_code", scope="openid email profile"
```

**Logout:**

```
2026-03-19 21:56:54,733 DEBUG [org.keycloak.events] (executor-thread-14)
type="LOGOUT", realmName="master", clientId="security-admin-console",
userId="7ab6219a-9145-46c4-9a9f-e63618c7b5e5",
sessionId="wUNuKbNoAA8RY9uIuAIYNxBC",
ipAddress="172.18.0.1"
```

**Failed login attempts:**

```
2026-03-19 22:06:53,312 WARN  [org.keycloak.events] (executor-thread-15)
type="LOGIN_ERROR", realmName="master", clientId="security-admin-console",
userId="7ab6219a-9145-46c4-9a9f-e63618c7b5e5",
ipAddress="172.18.0.1",
error="invalid_user_credentials", username="ajamthe@gmail.com"

2026-03-19 22:07:00,644 WARN  [org.keycloak.events] (executor-thread-22)
type="LOGIN_ERROR", realmName="master", clientId="security-admin-console",
userId="7ab6219a-9145-46c4-9a9f-e63618c7b5e5",
ipAddress="172.18.0.1",
error="invalid_user_credentials", username="user"
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

If `ipAddress` in the logs matches the nginx container IP, Keycloak is ignoring `X-Forwarded-For` and logging the proxy's IP. If it shows a different address, Keycloak is correctly reading the forwarded header.

---

## The PII Problem: Emails in the username Field

Look closely at the two failed login attempts above. The `username` field reflects exactly what the user typed:

- When a user logs in with their **email**, the log contains `username="ajamthe@gmail.com"`
- When a user logs in with a **username**, the log contains `username="user"`

This means **email addresses end up in your server logs** whenever users authenticate with their email. That's PII, and depending on your jurisdiction (GDPR, HIPAA, etc.) it may need to be handled carefully.

Here are three approaches to address it.

### Option 1: Custom Event Listener SPI (mask at source)

Write a small Keycloak provider in Java that intercepts events before they're logged and replaces the `username` value with a hash or redacted string — for example `a***@gmail.com` or `sha256:3a7bd3e2...`. This is the cleanest approach: PII never touches the log file at all.

This requires implementing Keycloak's `EventListenerProvider` SPI, packaging it as a JAR, and mounting it into the container under `/opt/keycloak/providers`.

### Option 2: Log scrubbing at the infrastructure layer

Use a log shipper — Fluentd, Logstash, or Vector — with a regex filter to redact email patterns before logs are stored or forwarded. For example, a Vector remap stage:

```toml
[transforms.redact_pii]
type = "remap"
inputs = ["keycloak_logs"]
source = '''
.message = replace(.message, r'username="[^"]*@[^"]*"', "username=\"[REDACTED]\"")
'''
```

The raw container log still contains the email, but it never reaches your log store or alerting pipeline.

### Option 3: Rely on userId instead of username

The `userId` field (`7ab6219a-9145-46c4-9a9f-e63618c7b5e5`) is already a non-PII UUID that uniquely identifies the user across all events. If you control how logs are queried and correlated, you can build your audit trail on `userId` alone and simply not index or store the `username` field.

No code required — but the email is still physically written to the container log, so this is a policy control rather than a technical one.

---

## Summary

| What | How |
|---|---|
| Enable event logging | Add `jboss-logging` listener in Admin Console |
| Surface events in server log | `KC_LOG_LEVEL=INFO,org.keycloak.events:DEBUG` |
| Forward real IP via nginx | `X-Forwarded-For`, `X-Real-IP` headers in nginx config |
| Tell Keycloak to trust the proxy | `KC_PROXY_HEADERS=xforwarded` |
| Prevent email PII in logs | Custom SPI, log scrubbing pipeline, or userId-based correlation |

Three environment variables and one nginx config block get you accurate, trustworthy login audit logs. Just make sure you account for the `username` field — it will carry your users' email addresses if that is how they authenticate.
