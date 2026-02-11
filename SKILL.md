---
name: matrix-homeserver-setup
description: A practical guide for installing, configuring, and maintaining a production-ready Matrix homeserver on a low-cost VPS with PostgreSQL, Let's Encrypt TLS, and End-to-End Encryption (E2EE) defaults.
license: MIT
compatibility: Requires a VPS with Debian 13 or Ubuntu 24.04 LTS, root access, and a valid domain name.
---

# Matrix Homeserver Setup & Management

This skill guides you through hosting a Matrix homeserver using Synapse on a low-cost VPS. It focuses on a secure production setup with automatic TLS (Let's Encrypt), PostgreSQL, and E2EE defaults.

## 1. Prerequisites & Sizing

### VPS Selection
Target a VPS with at least **2GB RAM** (minimum) or **4GB RAM** (recommended for federation bursts).
* **DigitalOcean:** Basic Droplet (2GB/1vCPU).
* **Hetzner:** CX23 (4GB RAM) (Check IPv4 pricing).
* **AWS Lightsail:** $12/mo bundle (2GB RAM).
* **Vultr:** Cloud Compute (2GB RAM).

**OS:** Use **Debian 13** or **Ubuntu 24.04 LTS**.

### Domain & DNS
Choose a `server_name` (e.g., `example.com` or `matrix.example.com`). This cannot be changed later.

**DNS Records:**
1.  **A Record:** `matrix.example.com` -> `<VPS_IPv4>`.
2.  **AAAA Record:** `matrix.example.com` -> `<VPS_IPv6>` (optional).
3.  **Delegation (Recommended):** Use `example.com` as your User ID domain (`@user:example.com`) but host on `matrix.example.com`.
    * Serve `https://example.com/.well-known/matrix/server` with content:
        `{"m.server": "matrix.example.com:443"}`

## 2. Server Hardening

Run as root to secure the host.

```bash
# 1. Update system and create admin user
apt update && apt -y upgrade
adduser vpsadmin
usermod -aG sudo vpsadmin

# 2. Configure Firewall (UFW)
apt -y install ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp   # SSH
ufw allow 80/tcp   # HTTP (Required for Certbot/Let's Encrypt)
ufw allow 443/tcp  # HTTPS (Client/Federation)
ufw allow 8448/tcp # Federation (Only if NOT delegating to 443)
ufw enable

# 3. Install Fail2ban
apt -y install fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
systemctl enable --now fail2ban

```

## 3. Database Setup (PostgreSQL)

Synapse requires PostgreSQL for production.

```bash
apt -y install postgresql
# Create user and db
sudo -u postgres createuser --pwprompt synapse_user
sudo -u postgres createdb --owner synapse_user synapse

```

## 4. Synapse Installation

Install using official packages.

```bash
apt install -y lsb-release wget apt-transport-https
wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg [https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg](https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg)
echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] [https://packages.matrix.org/debian/](https://packages.matrix.org/debian/) $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/matrix-org.list
apt update
apt install -y matrix-synapse-py3

```

### Configuration (`homeserver.yaml`)

Edit `/etc/matrix-synapse/homeserver.yaml`:

1. **Database:** Switch to `psycopg2` (Postgres) using the user/password created above.
2. **Listeners:** Bind HTTP to `127.0.0.1` port `8008` and set `x_forwarded: true`.
3. **Public Base URL:** Set `public_baseurl: "https://matrix.example.com"`.
4. **End-to-End Encryption (E2EE):**
Enable encryption by default for new rooms. Note: Once enabled for a room, it cannot be disabled.
```yaml
# Options: 'all', 'invite', 'off'. 'invite' is standard for private chats.
encryption_enabled_by_default_for_room_type: invite

```



## 5. Reverse Proxy & Let's Encrypt (TLS)

Use Nginx to handle TLS.

**1. Install Nginx:**

```bash
apt -y install nginx

```

**2. Configure Nginx (`/etc/nginx/sites-available/matrix.conf`):**

* Listen on 443 and 8448 (if using 8448).
* Proxy `/_matrix` and `/_synapse/client` to `http://localhost:8008`.
* *Important:* Do not use trailing slashes in `proxy_pass`.

**3. Auto-TLS with Certbot:**
This automatically obtains and installs Let's Encrypt certificates.

```bash
# Install Certbot and Nginx plugin
apt -y install certbot python3-certbot-nginx

# Request Certificate (Follow prompts to redirect HTTP->HTTPS)
certbot --nginx -d matrix.example.com

# Test Auto-Renewal
certbot renew --dry-run

```

## 6. Finalize & Verify

1. **Start Synapse:** `systemctl start matrix-synapse`
2. **Check Federation:**
```bash
curl -sS [https://matrix.example.com:8448/_matrix/key/v2/server](https://matrix.example.com:8448/_matrix/key/v2/server)

```


3. **Backup Keys:** Immediately back up `/etc/matrix-synapse/signing.key`. If lost, you cannot migrate or restore your federation identity.

## 7. Troubleshooting

* **Signature Verification Failed:** Often caused by Nginx modifying the URL path. Ensure `proxy_pass` is clean.
* **E2EE Issues:** Ensure users verify their sessions. Check `encryption_enabled_by_default_for_room_type` is set.
* **Database:** If moving from SQLite to Postgres later, use `synapse_port_db`.

```
