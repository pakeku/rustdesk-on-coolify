# BetterDesk on Coolify (clean install)

This is the path when you want a **real web portal** plus RustDesk-compatible signal/relay.
[BetterDesk](https://github.com/UNITRONIX/BetterDesk) replaces official OSS `hbbs`/`hbbr` with one container (Go server + Node console).

Use **`betterdesk-docker-compose.yaml`** in Coolify.

---

## What you will have

| What | URL / port | Notes |
|------|------------|--------|
| **Web portal** (manage everything) | `http://betterdesk.10.0.0.57.sslip.io` | Coolify Traefik → container `:5000` |
| **Client API** | `http://10.0.0.57:21121` | Used in RustDesk “API Server” field |
| **ID / signal** | `10.0.0.57:21116` (TCP+UDP) | Or hostname with `:21116` |
| **Relay** | `10.0.0.57:21117` | Must match `RELAY_SERVERS` |

Replace `10.0.0.57` with your Coolify host IP if different.

```
Browser ──HTTP──► betterdesk.<ip>.sslip.io ──Traefik──► :5000  (console)
RustDesk clients ──TCP/UDP──► host :21115–21119, :21121       (protocol + API)
```

---

## 0. Nuke old official RustDesk (OSS)

In Coolify:

1. Open the old **RustDesk** resource.
2. **Stop** it, then **delete** the resource (and volumes if offered — you have no devices yet, so wipe is fine).
3. Confirm ports are free on the host:

```bash
ss -tulpn | grep -E '2111[4-9]|5000' || echo "ports free"
```

Nothing should still listen on `21115–21119` / old RustDesk.

---

## 1. Create Coolify resource

1. **+ New Resource** → **Docker Compose Empty** (not “Raw Compose”).
2. Name it e.g. `betterdesk`.
3. Paste the full contents of **`betterdesk-docker-compose.yaml`** from this repo.
4. Save.

---

## 2. Domain (reverse proxy for the portal)

On the **betterdesk** service → **Domains**:

```text
betterdesk.10.0.0.57.sslip.io
```

- Scheme: **HTTP** is fine on LAN (or enable HTTPS if you have it).
- Coolify injects `SERVICE_URL_BETTERDESK_5000` and routes to port **5000**.

That is the only URL you open in a **browser**.

---

## 3. Environment variables

In **Coolify → Environment Variables**, set at least:

```bash
# Required: address clients use for relay (host IP, not a Docker internal IP)
RELAY_SERVERS=10.0.0.57:21117

# First admin login (change after first login)
ADMIN_USERNAME=admin
ADMIN_PASSWORD=ChangeMe_StrongPassword_123

# Lab-friendly: allow devices without manual approval
ENROLLMENT_MODE=open

# Optional pin — use a tag that exists on ghcr.io (see below)
BETTERDESK_IMAGE_TAG=3.3.2
```

> **Image tags:** Upstream changelog versions are often ahead of what is published to GHCR.
> Confirmed pullable tags include: `latest`, `3.3.2`, `3.3.0`, `3.2.0`, `3.1.0`, `3.0.0`.
> If Coolify says `not found`, switch `BETTERDESK_IMAGE_TAG` (or the compose default) to one of those.

| Variable | Purpose |
|----------|---------|
| `RELAY_SERVERS` | **Required.** What clients are told for relay. Wrong value = register works, sessions fail. |
| `ADMIN_PASSWORD` | Seeds admin **only on first empty volume**. |
| `ENROLLMENT_MODE=open` | Skip “approve device” friction while learning. Switch to `managed` later if you want. |

---

## 4. Firewall / host ports

On `10.0.0.57` (and any cloud SG):

```text
5000/tcp   # optional if you only use Traefik on 80
21115/tcp
21116/tcp
21116/udp  # required
21117/tcp
21118/tcp  # optional web client
21119/tcp  # optional web client
21121/tcp  # client API
```

Coolify’s proxy already handles **80** for `betterdesk.…sslip.io`.

---

## 5. Deploy

Click **Deploy** in Coolify. Wait until the container is **healthy** (can take ~1 minute).

### Get admin password (if you did not set `ADMIN_PASSWORD`)

On the Coolify host:

```bash
docker ps | grep -i betterdesk
docker exec <container-name> betterdesk-show-admin-credentials
```

### Health checks

```bash
curl -sS http://10.0.0.57:21121/api/health
curl -sS -o /dev/null -w "%{http_code}\n" http://betterdesk.10.0.0.57.sslip.io/
```

Expect API JSON health and portal **200** (or redirect to `/login`).

---

## 6. Open the portal

Browser:

```text
http://betterdesk.10.0.0.57.sslip.io
```

1. Log in: `admin` / your `ADMIN_PASSWORD`.
2. **Change the password** immediately.
3. Find the **public key** (Dashboard / Settings / Server keys — UI labels vary by version).
4. Note the key string (same idea as OSS `id_ed25519.pub`).

---

## 7. Configure the first RustDesk **client**

Install [RustDesk](https://rustdesk.com/) on a PC, then **Settings → Network → ID/Relay Server**:

| Field | Value (your LAN) |
|-------|------------------|
| **ID Server** | `10.0.0.57:21116` or `betterdesk.10.0.0.57.sslip.io:21116` |
| **Relay Server** | `10.0.0.57:21117` |
| **API Server** | `http://10.0.0.57:21121` |
| **Key** | public key from the portal |

Fully **restart** the RustDesk client.

Optional: click the **account** icon and log in with the same BetterDesk user for address book / “Pro-like” features.

Status should become **Ready**. A second machine with the same server settings can connect by ID.

---

## 8. Mental model (so it stays clear)

| Interface | Role |
|-----------|------|
| **`http://betterdesk.…sslip.io`** | Admin portal — users, devices, keys, policies |
| **RustDesk desktop/mobile app** | Actual remote desktop sessions |
| **Coolify** | Deploy, logs, domain, env, restarts |

You do **not** manage remote desktops only in Coolify — Coolify runs BetterDesk; BetterDesk portal + clients do the rest.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Portal 502 | Domain not on port 5000; container not healthy; old stack still bound |
| API health fails | Port `21121` not published/open; wait for start_period |
| Client “Ready” but no connect | `RELAY_SERVERS` must be **host** IP/DNS `:21117`, not `172.x` |
| Key mismatch | Copy key from **this** BetterDesk instance only |
| Device not listed / blocked | `ENROLLMENT_MODE=managed` → approve in portal, or set `open` and redeploy (fresh policy) |
| Forgot admin password | Reset via portal tools, or new volumes + `ADMIN_PASSWORD` (destroys data) |

```bash
# Useful on the Coolify host
docker logs <betterdesk-container> --tail 100
curl -sS http://127.0.0.1:21121/api/health
ss -tulpn | grep -E '2111|5000'
```

---

## Optional next steps

- Pin `BETTERDESK_IMAGE_TAG` and upgrade intentionally.
- Set `ENROLLMENT_MODE=managed` after the lab works.
- Put the API behind HTTPS (separate domain or reverse proxy to `21121`) for WAN clients.
- Read upstream: [BetterDesk](https://github.com/UNITRONIX/BetterDesk) · [Docker quickstart](https://github.com/UNITRONIX/BetterDesk/blob/main/docs/docker/DOCKER_QUICKSTART.md)

BetterDesk is **AGPL-3.0**, third-party (not RustDesk Inc.). Fine for homelab; review license for commercial use.
