# DigitalOcean (UK) deployment (Docker + Caddy)

This deploys Obot on a single DigitalOcean Droplet (London region) using Docker Compose, with Caddy providing automatic HTTPS.

## 0) Put this repo in your GitHub

If you want to keep your own deployment config (and optionally use GitHub Actions), fork Obot:

```bash
gh repo fork obot-platform/obot --clone=false
git remote rename origin upstream
git remote add origin https://github.com/<your-gh-org-or-user>/obot.git
git push -u origin main
```

## 1) Create a droplet (London / UK)

Recommended minimum: **2 GB RAM** (Obot runs a database + background workers + tools).

Via DO UI:
- Region: **London (lon1)**
- Image: **Ubuntu 24.04 LTS** (or 22.04)
- Size: Basic `s-1vcpu-2gb` (or larger)
- Add your SSH key

## 2) Bootstrap the server

SSH in, then:

```bash
sudo bash -lc 'apt-get update && apt-get install -y git'
git clone https://github.com/<your-gh-org-or-user>/obot.git
cd obot/deploy
sudo bash ./server-bootstrap-ubuntu.sh
```

This installs Docker + Compose, enables the daemon, and opens ports `80/443` in UFW.

## 3) Point a domain at the droplet

Create an `A` record to the droplet’s public IPv4:
- `obot.yourdomain.com` → `<DROPLET_IP>`

### No domain? Use `nip.io` / `sslip.io` (works with HTTPS)

If you don’t want to use your own domain, you can use a free “IP-in-hostname” domain such as:
- `<DROPLET_IP>.nip.io`
- `<DROPLET_IP>.sslip.io`

These resolve to your droplet IP without DNS setup, and you can still get HTTPS certificates via Let’s Encrypt (HTTP-01) as long as ports `80/443` are reachable.

## 4) Configure runtime secrets on the droplet

On the droplet:

```bash
sudo mkdir -p /opt/obot
sudo cp docker-compose.yml Caddyfile /opt/obot/
sudo cp env.example /opt/obot/.env
sudo nano /opt/obot/.env
```

### Optional: add a username/password prompt (Basic Auth)

If you want a simple username/password prompt in front of Obot (in addition to GitHub/Google login inside Obot):

```bash
sudo cp Caddyfile.basic-auth /opt/obot/Caddyfile
docker run --rm caddy:2.8-alpine caddy hash-password --plaintext 'your-password'
```

Then set `OBOT_BASIC_AUTH_USER` + `OBOT_BASIC_AUTH_HASH` in `/opt/obot/.env`.

At minimum set:
- `OBOT_DOMAIN=obot.yourdomain.com` (or `<DROPLET_IP>.nip.io`)
- `OBOT_SERVER_HOSTNAME=obot.yourdomain.com` (or `<DROPLET_IP>.nip.io`)
- `OPENAI_API_KEY=...`
- `OBOT_BOOTSTRAP_TOKEN=...` (pick a long random string)
- `OBOT_SERVER_ENABLE_AUTHENTICATION=true` (recommended)
- `OBOT_SERVER_AUTH_OWNER_EMAILS=you@yourdomain.com` (recommended)

## 5) Start Obot

```bash
cd /opt/obot
sudo docker compose pull
sudo docker compose up -d
```

Visit:
- `https://obot.yourdomain.com`
- Admin: `https://obot.yourdomain.com/admin`

## 6) Configure Google login

In Obot:
- Admin → **Auth Providers** → **Google** → **Configure**
- Copy the **Redirect/Callback URL** shown there.

In Google Cloud Console:
- APIs & Services → Credentials → Create Credentials → **OAuth client ID**
- Application type: **Web application**
- Add the redirect URL you copied from Obot
- Paste Client ID/Secret back into Obot.

## 6b) Configure GitHub login (instead of Google)

In Obot:
- Admin → **Auth Providers** → **GitHub** → **Configure**
- Copy the **Redirect/Callback URL** shown there.

In GitHub:
- Settings → Developer settings → OAuth Apps → **New OAuth App**
- Homepage URL: `https://<your-host>` (for `nip.io`: `https://<DROPLET_IP>.nip.io`)
- Authorization callback URL: paste the URL from Obot
- Copy **Client ID** and generate a **Client Secret**
- Paste Client ID/Secret back into Obot.

## Updating

```bash
cd /opt/obot
sudo docker compose pull
sudo docker compose up -d
```

## Optional: GitHub Actions deploy

This repo includes `.github/workflows/deploy-digitalocean.yml` which pushes `deploy/docker-compose.yml`, `deploy/Caddyfile`, and an `.env` to your droplet, then runs `docker compose up -d`.

Create these GitHub repo secrets:
- `DO_HOST`: droplet IP or hostname
- `DO_USER`: usually `root`
- `DO_SSH_PRIVATE_KEY`: private key for SSH
- `DO_APP_DIR`: optional, defaults to `/opt/obot`
- `DO_USE_BASIC_AUTH`: optional, set to `true` to deploy `Caddyfile.basic-auth` as `Caddyfile`
- `OBOT_ENV`: full contents of the droplet `.env` file (use `deploy/env.example` as a base)
