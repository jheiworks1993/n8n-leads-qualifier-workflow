# n8n Automation — Portfolio Project

Self-hosted [n8n](https://n8n.io) workflow automation running on a VPS with Nginx reverse proxy and SSL via Let's Encrypt.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [VPS Setup](#vps-setup)
3. [Install Node.js](#install-nodejs)
4. [Install n8n](#install-n8n)
5. [Run n8n as a Systemd Service](#run-n8n-as-a-systemd-service)
6. [Install and Configure Nginx](#install-and-configure-nginx)
7. [SSL with Let's Encrypt (Certbot)](#ssl-with-lets-encrypt-certbot)
8. [Environment Variables](#environment-variables)
9. [Firewall Rules](#firewall-rules)
10. [Updating n8n](#updating-n8n)
11. [Useful Commands](#useful-commands)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- A VPS (Ubuntu 22.04 LTS recommended) with root or sudo access
- A domain name pointed to the VPS IP via an A record (e.g., `n8n.yourdomain.com`)
- SSH access to the server

---

## VPS Setup

### 1. Connect and update

```bash
ssh root@YOUR_VPS_IP
apt update && apt upgrade -y
```

### 2. Create a non-root user (recommended)

```bash
adduser n8nuser
usermod -aG sudo n8nuser
su - n8nuser
```

### 3. Set the hostname (optional but clean)

```bash
hostnamectl set-hostname n8n-server
```

---

## Install Node.js

n8n requires Node.js 18 or later. Use NodeSource for a clean install:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # should print v20.x.x
npm -v
```

---

## Install n8n

Install n8n globally via npm:

```bash
sudo npm install -g n8n
```

Verify the installation:

```bash
n8n --version
```

---

## Run n8n as a Systemd Service

Running n8n as a service ensures it starts on boot and restarts on failure.

### 1. Create the service file

```bash
sudo nano /etc/systemd/system/n8n.service
```

Paste the following (replace `n8nuser` with your actual user):

```ini
[Unit]
Description=n8n Workflow Automation
After=network.target

[Service]
Type=simple
User=n8nuser
Environment=N8N_HOST=0.0.0.0
Environment=N8N_PORT=5678
Environment=N8N_PROTOCOL=https
Environment=WEBHOOK_URL=https://n8n.yourdomain.com/
Environment=N8N_BASIC_AUTH_ACTIVE=true
Environment=N8N_BASIC_AUTH_USER=admin
Environment=N8N_BASIC_AUTH_PASSWORD=your_secure_password
Environment=NODE_ENV=production
ExecStart=/usr/bin/n8n start
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> Tip: Move sensitive values to a `.env` file and use `EnvironmentFile=/home/n8nuser/.n8n/.env` instead of inline `Environment=` lines.

### 2. Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable n8n
sudo systemctl start n8n
sudo systemctl status n8n
```

---

## Install and Configure Nginx

### 1. Install Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 2. Create the n8n site config

```bash
sudo nano /etc/nginx/sites-available/n8n
```

Paste:

```nginx
server {
    listen 80;
    server_name n8n.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
}
```

### 3. Enable the site

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t          # test config
sudo systemctl reload nginx
```

---

## SSL with Let's Encrypt (Certbot)

### 1. Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 2. Obtain and install the certificate

```bash
sudo certbot --nginx -d n8n.yourdomain.com
```

Follow the prompts. Certbot will automatically update your Nginx config to redirect HTTP → HTTPS.

### 3. Verify auto-renewal

```bash
sudo certbot renew --dry-run
```

Certbot installs a systemd timer that handles renewal automatically. Check it with:

```bash
systemctl status certbot.timer
```

---

## Environment Variables

Key variables you may want to configure (set in the systemd service or a `.env` file):

| Variable | Description | Example |
|---|---|---|
| `N8N_HOST` | Host n8n listens on | `0.0.0.0` |
| `N8N_PORT` | Port n8n listens on | `5678` |
| `N8N_PROTOCOL` | Protocol (http/https) | `https` |
| `WEBHOOK_URL` | Public URL for webhooks | `https://n8n.yourdomain.com/` |
| `N8N_BASIC_AUTH_ACTIVE` | Enable basic auth | `true` |
| `N8N_BASIC_AUTH_USER` | Basic auth username | `admin` |
| `N8N_BASIC_AUTH_PASSWORD` | Basic auth password | `your_password` |
| `N8N_ENCRYPTION_KEY` | Key to encrypt credentials | `a_random_32char_string` |
| `DB_TYPE` | Database type | `sqlite` (default) or `postgresdb` |
| `EXECUTIONS_PROCESS` | Execution mode | `main` or `own` |
| `N8N_LOG_LEVEL` | Logging verbosity | `info` |

---

## Firewall Rules

Allow only necessary ports using UFW:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'   # opens port 80 and 443
sudo ufw enable
sudo ufw status
```

> Do NOT expose port 5678 publicly — all traffic should go through Nginx.

---

## Updating n8n

```bash
sudo npm install -g n8n@latest
sudo systemctl restart n8n
n8n --version   # confirm new version
```

Always check the [n8n changelog](https://github.com/n8n-io/n8n/releases) before upgrading in production — breaking changes do happen.

---

## Useful Commands

```bash
# Service management
sudo systemctl status n8n
sudo systemctl restart n8n
sudo systemctl stop n8n

# View n8n logs
sudo journalctl -u n8n -f
sudo journalctl -u n8n --since "1 hour ago"

# Nginx
sudo nginx -t                   # test config syntax
sudo systemctl reload nginx     # apply config changes
sudo tail -f /var/log/nginx/error.log

# SSL certificate info
sudo certbot certificates
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| n8n not starting | Port 5678 already in use | `lsof -i :5678` and kill the process |
| 502 Bad Gateway | n8n service down | `systemctl restart n8n` |
| Webhooks not triggering | `WEBHOOK_URL` misconfigured | Ensure it matches your public domain |
| SSL certificate errors | Domain DNS not propagated | Wait and re-run `certbot --nginx` |
| Can't login | Basic auth not set | Check env vars in the service file |
| WebSocket errors | Missing proxy headers | Verify `Upgrade` and `Connection` headers in Nginx config |

---

## Project Structure

```
n8n-automation/
├── AI Leads Qualifier Demo.json   # Exported n8n workflow
├── keys/                          # API keys / credentials (gitignored)
├── secret.json                    # Secrets file (gitignored)
├── package.json
└── README.md
```

---

> Built as a portfolio project demonstrating self-hosted workflow automation.
