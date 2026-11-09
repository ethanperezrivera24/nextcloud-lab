# Self-Hosted Nextcloud over Tailscale

A self-hosted Google Drive / iCloud alternative: **Nextcloud** running in Docker on an Ubuntu 26.04 cloud VM, reachable **only** through a private **Tailscale** network, with a real HTTPS certificate. No ports are open to the public internet.

Built as a homelab learning project to get hands-on with containers, private networking, reverse proxies, and basic server hardening.

---

## Architecture

```
 iPhone / Laptop (Tailscale client)
            │
            │  WireGuard-encrypted tailnet
            ▼
 ┌─────────────────── Ubuntu 26.04 VM ───────────────────┐
 │                                                        │
 │  Tailscale Serve  (HTTPS :443, Let's Encrypt cert)     │
 │        │                                               │
 │        ▼  proxy → 127.0.0.1:8080                       │
 │  ┌──────────── Docker Compose ─────────────┐           │
 │  │  app: nextcloud:stable  (Apache + PHP)  │           │
 │  │        │                                │           │
 │  │        ▼                                │           │
 │  │  db:  mariadb:11.4                      │           │
 │  └─────────────────────────────────────────┘           │
 │  Volumes: nextcloud (app + files), db (database)       │
 └────────────────────────────────────────────────────────┘
```

**Request path:** client → Tailscale tunnel → Tailscale Serve terminates TLS → plain HTTP to Nextcloud on localhost → Nextcloud talks to MariaDB on the internal Docker network.

## Tech Stack

| Component | Role |
|---|---|
| Ubuntu Server 26.04 | Host OS (2 vCPU, 4 GB RAM, 50 GB disk) |
| Docker + Docker Compose | Runs Nextcloud and MariaDB as containers |
| Nextcloud 34 | File sync, web UI, iOS app backend |
| MariaDB 11.4 | Nextcloud's database |
| Tailscale | Private mesh VPN (WireGuard) + MagicDNS |
| Tailscale Serve | Reverse proxy + automatic HTTPS certificate |

## Security Design

- **Zero public exposure.** The server has no port forwarding and no public-facing web port. Only devices authenticated to my tailnet can reach it.
- **Port bound to localhost.** Nextcloud publishes `127.0.0.1:8080` only, so it isn't reachable even from the VM's local network. All traffic has to come through Tailscale Serve.
- **Real TLS.** Tailscale issues a valid certificate for the `*.ts.net` hostname, so clients get proper HTTPS with no self-signed warnings. The iOS app's "fall back to insecure HTTP" prompt is always declined.
- **Isolated from my lab network.** Nextcloud runs on its own VM, separate from my pfSense attack/defense lab, so breaking the lab never takes down my files.
- **Secrets out of version control.** DB credentials live in a `.env` file (gitignored), generated with `openssl rand -base64 24`.
- **Patched host.** Applied all pending Ubuntu security updates after setup.
- **Key expiry disabled for the server node only**, so the server doesn't silently drop off the tailnet after 180 days. Client devices keep expiry on.

**Threat model note:** Because the login page is only reachable from inside the tailnet, the real front door is the Tailscale identity (a Google account protected with 2FA), not the Nextcloud password. Files cached on the phone by the Nextcloud app sit outside Tailscale's protection, so the phone passcode and the app's own Face ID lock matter.

## Setup

### 1. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # log out/in afterward
docker run hello-world
```

### 2. Join the tailnet
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```
In the Tailscale admin console: enable **MagicDNS** and **HTTPS Certificates**.

### 3. Configure and start Nextcloud
```bash
mkdir ~/nextcloud && cd ~/nextcloud
cp .env.example .env && nano .env   # fill in generated passwords
chmod 600 .env
docker compose up -d
docker compose logs -f app
```
See [`compose.yaml`](compose.yaml).

### 4. Put HTTPS in front with Tailscale Serve
```bash
sudo tailscale serve --bg 8080
tailscale serve status
```

### 5. Tell Nextcloud it lives behind an HTTPS proxy
```bash
OCC="docker compose exec -u www-data app php occ"
$OCC config:system:set trusted_domains 1 --value=nextcloud.<tailnet>.ts.net
$OCC config:system:set overwriteprotocol --value=https
$OCC config:system:set overwrite.cli.url --value=https://nextcloud.<tailnet>.ts.net
$OCC config:system:set trusted_proxies 0 --value=172.18.0.1
```

### 6. Connect clients
Log in from the browser and the Nextcloud iOS app using the **base URL** `https://nextcloud.<tailnet>.ts.net`. Auto Upload is configured for photos only, to fit the disk budget.

## Troubleshooting Log

Real problems I hit and how I solved them:

| Problem | Cause | Fix |
|---|---|---|
| `sudo: preserving the entire environment is not supported, '-E' is ignored` during Docker install | Ubuntu 26.04 replaced classic sudo with **sudo-rs** (a Rust rewrite), which doesn't support `-E` | Harmless. The install doesn't need it |
| "Access through untrusted domain" on the phone | `NEXTCLOUD_TRUSTED_DOMAINS` is only applied during *automated* install. I finished setup in the browser, so only the setup hostname was trusted | Added domains at runtime with `occ config:system:set trusted_domains` |
| `docker compose exec` → "no configuration file provided" | Compose looks for `compose.yaml` in the current directory | `cd ~/nextcloud` first |
| iOS app: "secure connection could not be established" → "hostname could not be found" | I entered `https://…/login`. The app appends its own paths (`/status.php`) to the base URL | Use only the base URL, no path |
| Login loops / wrong links behind the proxy | Nextcloud only sees plain HTTP from the proxy | Set `overwriteprotocol=https` and `trusted_proxies` |
| 31 GB photo library vs. 22 GB free | VM disk is fixed and can't be expanded | Auto-uploaded photos only (no videos) and monitored usage with `df -h /` |
| App shows files with Tailscale off | That's the app's local cache, not a live connection | Expected behavior. Turned on the app's passcode lock |

## What I Learned

- Running multi-container apps with **Docker Compose**: services, named volumes, internal networks, restart policies
- Using **Tailscale** for private access instead of port forwarding, which matters when you don't control the router
- How a **TLS-terminating reverse proxy** works, and why the backend app needs `overwriteprotocol` / `trusted_proxies` to generate correct URLs
- Using Nextcloud's `occ` admin CLI inside a container as the correct user (`-u www-data`)
- **Certificate Transparency**: every public cert (including my `ts.net` hostname) is publicly logged, which attackers use for recon
- Keeping "infrastructure I experiment on" separate from "infrastructure I depend on"
- `curl | sh` installers run whatever the URL returns, so only use trusted sources

## Limitations & Next Steps

This is a learning deployment, not yet a replacement for iCloud/Google Drive:

- [ ] **Backups.** Right now there's a single copy on borrowed infrastructure. Next step: automated volume + DB dumps following the 3-2-1 rule, plus a tested restore.
- [ ] **Dedicated hardware.** A used mini PC with a large external drive, so I control the disk and the uptime.
- [ ] **Background jobs.** Switch Nextcloud from AJAX to system cron.
- [ ] **Monitoring.** Disk usage alerts and container health checks.
- [ ] **Nextcloud 2FA (TOTP)** as defense in depth.
