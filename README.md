# RustDesk on Coolify

Self-host **RustDesk Server** (`hbbs` + `hbbr`) on your own infrastructure using **Coolify**, with Docker, host-published TCP/UDP ports, and a Coolify-managed domain.

This repository provides a **Coolify-ready `rustdesk-docker-compose.yaml`** that lets Coolify manage domains and env injection, while publishing the ports RustDesk needs. It addresses [Coolify discussion #9453](https://github.com/coollabsio/coolify/discussions/9453).

## 🚀 Features

- Open-source remote desktop server (RustDesk OSS)
- Coolify-native deployment (magic `SERVICE_*` variables)
- Single supervised container (`rustdesk-server-s6`) running **hbbs** + **hbbr**
- Real Docker health checks via `/usr/bin/healthcheck.sh`
- Domain assignment in Coolify (your domain or IP-based wildcard DNS)
- Host-published ports for NAT hole-punching and relay
- Persistent storage for server keys and data
- Production-ready defaults

## 📦 Requirements

- A server with **Coolify** installed
- A hostname pointing to your Coolify server, for example:
  - Production: `rustdesk.yourdomain.com`
  - Lab / no custom DNS: `rustdesk.<your-server-ip>.sslip.io`
- Docker (managed by Coolify)
- Firewall / security group rules allowing the RustDesk ports below

## 🧱 Stack

- **RustDesk Server (s6)** — `rustdesk/rustdesk-server-s6`
  - **hbbs** — ID / rendezvous server
  - **hbbr** — relay server
- **Coolify** — domain, env, deploy orchestration
- **Traefik** (via Coolify) — optional web-client websocket paths only

> **Important:** The core RustDesk protocol is **raw TCP/UDP**, not HTTP. Coolify’s reverse proxy alone is not enough. Ports must be published on the host and opened in the firewall. Opening `rustdesk.yourdomain.com` in a browser will not show a web app.

## 📁 Repository Structure

```
.
├── rustdesk-docker-compose.yaml
├── README.md
├── index.html
├── LICENSE
└── .github/workflows/jekyll-gh-pages.yml
```

## 🔌 Required Ports

| Port | Protocol | Service | Purpose |
|------|----------|---------|---------|
| 21115 | TCP | hbbs | NAT type test |
| 21116 | TCP + **UDP** | hbbs | ID registration, heartbeat, hole punching |
| 21117 | TCP | hbbr | Relay |
| 21118 | TCP | hbbs | Web client (optional) |
| 21119 | TCP | hbbr | Web client (optional) |

If you use Cloudflare, set the RustDesk hostname to **DNS only** (grey cloud). Do not proxy raw TCP/UDP through Cloudflare’s HTTP proxy.

## ⚙️ Deployment (Coolify)

### 1. Create a new Resource

- Application type: **Docker Compose Empty**
- Copy + paste the contents of `rustdesk-docker-compose.yaml`

Do **not** use “Raw Compose Deployment” if you want Coolify’s `SERVICE_*` magic variables to work.

### 2. Configure Domain

In **Coolify → Domains** (for the `rustdesk` service):

- Add your hostname, for example:

  ```
  rustdesk.yourdomain.com
  ```

- Or, for lab setups without a real domain (same idea as other Coolify apps on `*.sslip.io`):

  ```
  rustdesk.<your-server-ip>.sslip.io
  ```

  Replace `<your-server-ip>` with the public (or LAN) IP of your Coolify host, e.g. if the server is `203.0.113.10`:

  ```
  rustdesk.203.0.113.10.sslip.io
  ```

Coolify will inject:

- `SERVICE_FQDN_RUSTDESK` → used as the public hostname for clients and for `RELAY=`
- Optional `SERVICE_URL_RUSTDESK_21118` / `21119` for web-client websocket paths

Use that hostname in the **RustDesk client** (ID / Relay). Main traffic uses published ports **21116** / **21117**, not HTTPS on 443.

### 3. Environment Variables

In **Coolify → Configure → Environment Variables**, optional overrides:

```
RUSTDESK_VERSION=latest
ALWAYS_USE_RELAY=N
ENCRYPTED_ONLY=0
RUST_LOG=info

# Only if you need non-default host port mappings:
RUSTDESK_HBBS_NAT_PORT=21115
RUSTDESK_HBBS_PORT=21116
RUSTDESK_HBBR_PORT=21117
RUSTDESK_HBBS_WS_PORT=21118
RUSTDESK_HBBR_WS_PORT=21119
```

`RELAY` is set automatically from `SERVICE_FQDN_RUSTDESK` and `RUSTDESK_HBBR_PORT` in the compose file. Set the Coolify domain before or when you deploy so `hbbs` gets the correct relay hostname.

### 4. Open firewall ports

On the host and/or cloud provider:

```
21115/tcp
21116/tcp
21116/udp
21117/tcp
21118/tcp   # optional, web client
21119/tcp   # optional, web client
```

### 5. Deploy

Click **Deploy** in Coolify.

Initial startup is usually quick. Keys are generated on first run and stored in the `rustdesk-data` volume.

## 🩺 Health Checks

The s6 image includes a built-in health script (as noted in [discussion #9453](https://github.com/coollabsio/coolify/discussions/9453)):

```yaml
healthcheck:
  test:
    - CMD
    - /usr/bin/healthcheck.sh
  interval: 10s
  timeout: 5s
  retries: 3
  start_period: 10s
```

It verifies that both **hbbr** and **hbbs** are running under s6. Coolify uses the container health status for readiness.

## 🔑 Get the public key

After deploy, grab the key for the RustDesk client.

From Coolify logs (rustdesk service), look for:

```
Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
```

Or copy the key file from the container:

```bash
docker cp <rustdesk-container-name>:/data/id_ed25519.pub ./id_ed25519.pub
cat ./id_ed25519.pub
```

> Prefer logs or `docker cp` if `docker exec` fails on minimal images. This compose uses the **s6** image so health checks work reliably.

## 📱 RustDesk client settings

| Field | Value |
|-------|--------|
| **ID Server** | `rustdesk.yourdomain.com:21116` |
| **Relay Server** | `rustdesk.yourdomain.com:21117` |
| **API Server** | leave empty |
| **Key** | public key from logs / `id_ed25519.pub` |

Using explicit ports in the client often helps during setup. Fully restart the client after changing servers.

Expected client status when healthy: **Ready**.

## 🔐 Security Notes

- Treat the server **private key** as a secret; back up the volume, do not commit keys
- Prefer a domain you control in production; IP-based `sslip.io` names are fine for labs
- Restrict Coolify access to trusted users
- If you enable Cloudflare, keep the hostname **DNS-only** for RustDesk ports
- Set `ALWAYS_USE_RELAY=Y` if direct hole-punching fails in your network (higher relay load)

## 🛠 Customization

Common tweaks:

- Pin `RUSTDESK_VERSION` to a specific tag instead of `latest`
- Remap host ports if 21115–21119 are already in use
- Force relay with `ALWAYS_USE_RELAY=Y`
- Disable web-client ports (21118/21119) in firewall if unused

## 🧩 Troubleshooting

### Container unhealthy / restarting

- Confirm the image is `rustdesk/rustdesk-server-s6` (includes `/usr/bin/healthcheck.sh`)
- Check Coolify logs for hbbs/hbbr startup errors
- Ensure the data volume is writable

### Client stuck on “Connecting to the RustDesk network…”

- Verify **UDP 21116** is open (TCP-only tests are not enough)
- Confirm ID server, relay server, and **key** match the server
- Fully restart the RustDesk client after config changes
- Try explicit ports: `your-host:21116` and `your-host:21117`

### Ports not listening

```bash
docker ps | grep rustdesk
ss -tulpn | grep 2111
```

Expect listeners on at least **21116/tcp**, **21116/udp**, and **21117/tcp**.

### Domain / Coolify

- Confirm the domain in Coolify matches what you enter in the client
- Confirm Coolify injected `SERVICE_FQDN_RUSTDESK`
- Redeploy after changing the domain so `RELAY=` updates

### TCP works but UDP does not

- `Test-NetConnection host -Port 21116` only checks TCP
- Check cloud security groups and `ufw` / iptables for **UDP 21116**

### Browser shows nothing

Expected. RustDesk Server OSS is not a website. Use the **RustDesk client**, not Chrome/Firefox, to connect.

## 📄 License

This project is licensed under the **MIT License**.

See the `LICENSE` file for details.

---

## 🙌 Credits

- [RustDesk](https://rustdesk.com/)
- [RustDesk Server](https://github.com/rustdesk/rustdesk-server)
- [Coolify](https://coolify.io/)
- Community work on [Coolify discussion #9453](https://github.com/coollabsio/coolify/discussions/9453)
