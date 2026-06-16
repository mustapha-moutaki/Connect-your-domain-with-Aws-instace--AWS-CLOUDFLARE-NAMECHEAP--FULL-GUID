# 🚀 AWS DEPLOYMENT GUIDE
### Beginner → Running Spring Boot with Docker + HTTPS + Cloudflare

---

## 🟢 STEP 1 — Create EC2 Instance
In AWS Console:

Go to:
👉 EC2 → Launch Instance

Choose:
- Name: `springboot-server`
- OS: `Ubuntu 22.04`
- Instance type: `t3.micro` (free tier)

### 🔐 Key Pair (VERY IMPORTANT)

Create or select:

👉 `springboot-key.pem`

📌 **Why we use this key?**

It is used for SSH authentication instead of password.

- ✔ safer
- ✔ encrypted login
- ✔ only your machine can access server

---

## 🟢 STEP 2 — Secure your key file (IMPORTANT)

Run this on your LOCAL machine:

```bash
chmod 400 ~/.ssh/springboot-key.pem
```

📌 **N.B (IMPORTANT EXPLANATION)**

We use `chmod 400`

Why?

It means:
- only YOU can read the file
- nobody else (including system users) can modify it
- SSH REQUIREMENT (otherwise it refuses connection)

👉 Without this command, SSH will fail.

---

## 🟢 STEP 3 — Connect to AWS server

```bash
ssh -i ~/.ssh/springboot-key.pem ubuntu@<INSTANCE_PUBLIC_IP>
```

Example:
```bash
ssh -i ~/.ssh/springboot-key.pem ubuntu@13.49.72.54
```

---

## 🟢 STEP 4 — Update server

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 🟢 STEP 5 — Install required tools

### ☕ Install Java (JDK)
```bash
sudo apt install openjdk-17-jdk -y
```

### 📦 Install Maven
```bash
sudo apt install maven -y
```

### 🐳 Install Docker
```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

Add user to docker group:
```bash
sudo usermod -aG docker ubuntu
```
👉 then logout & login again

### 🔧 Install Git
```bash
sudo apt install git -y
```

---

## 🟢 STEP 6 — Clone your project

With specific branch:
```bash
git clone -b feat/fix-migration-aws https://github.com/mustapha-moutaki/AlexIS-Alexsys-client-support--backend.git
```

---

## 🟢 STEP 7 — Go to project

```bash
cd AlexIS-Alexsys-client-support--backend
```

---

## 🟢 STEP 8 — Run with Docker

If you already have Dockerfile + docker-compose:
```bash
docker compose up --build -d
```

---

## 🟢 STEP 9 — Check running containers

```bash
docker ps
```

---

## 🟢 STEP 10 — Check logs (VERY IMPORTANT)

```bash
docker logs -f alexsis_app
```

---

## 🌐 STEP 11 — Access app

```
http://<YOUR_PUBLIC_IP>:8080
```

---

## 🚨 COMMON MISTAKES (YOU ALREADY FACED THESE)

### ❌ 1. Port not open
Fix in Security Group: `8080 open`

### ❌ 2. Key permission error
```bash
chmod 400 key.pem
```

### ❌ 3. Docker container exits
```bash
docker logs
```

### ❌ 4. Database error (Postgres)
Fix: `POSTGRES_PASSWORD required`

---

## 🧠 BIG PICTURE (UNDERSTAND THIS)

```
GitHub → EC2 → Docker → Spring Boot → PostgreSQL → Internet
```

---

## 🚀 NEXT LEVEL (WHEN YOU ARE READY)

### 🔥 1. Elastic IP (STATIC IP)

**Problem:** EC2 public IP changes when instance restarts

**Solution:**
- Allocate Elastic IP
- Attach to instance

- ✔ stable server IP
- ✔ required for production
- ✔ needed for domain later

---

### 🔥 2. Nginx Reverse Proxy (PRODUCTION STANDARD)

> Nginx is a reverse proxy — it is like an intermediate between the app and user, so it is also like an API gateway. It says: if the user visits this IP address, the port by default is 80 (80 is the default port of browsers), so if you visit this port it forwards you to the configured port of our app 8080.

Instead of:
```
http://IP:8080
```
We use:
```
http://IP
```

Install Nginx:
```bash
sudo apt install nginx -y
```

Config:
```bash
sudo nano /etc/nginx/sites-available/default
```

Add:
```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Restart:
```bash
sudo systemctl restart nginx
```

---

### 🔥 3. Security Group Update (IMPORTANT)

Add:

| Type | Port |
|------|------|
| HTTP | 80   |

---

### 🔥 4. DOMAIN NAME (PRO LEVEL)

After Elastic IP:
- Buy domain (Namecheap / Route53)
- Point DNS → Elastic IP

Example:
```
api.myapp.com → EC2 Elastic IP
```

---

### 🔥 5. HTTPS (SSL / HTTPS SECURITY)

Install Certbot:
```bash
sudo apt install certbot python3-certbot-nginx -y
```

Enable HTTPS:
```bash
sudo certbot --nginx
```

Result:
```
https://api.myapp.com
```
- ✔ encrypted
- ✔ production ready
- ✔ trusted by browsers

---

### 🔥 6. CI/CD (AUTOMATION)

Flow:
```
GitHub push → GitHub Actions → SSH EC2 → docker compose up
```
- ✔ no manual deployment
- ✔ professional workflow

---

## 🚀 FINAL PRO SYSTEM

```
User
  ↓
Domain (HTTPS)
  ↓
Elastic IP
  ↓
Nginx (80/443)
  ↓
Spring Boot (Docker 8080 internal)
  ↓
PostgreSQL (Docker)
```

---
---

# ❓ YOUR QUESTIONS — ANSWERED

---

### Q1 — What is `server_name` in the Nginx config file?

`server_name` tells Nginx which domain name this server block should respond to.

When a request comes in, Nginx looks at the `Host` header of the request and matches it against all `server_name` values. If it matches, that block handles the request.

Example:
```nginx
server_name moutaki.tech www.moutaki.tech;
```
This means: **only handle requests where the Host is `moutaki.tech` or `www.moutaki.tech`.**

If the Host does not match any block, Nginx uses the default block — which returns 403 or a blank page.

> 💡 That is exactly why you were getting 403 before — `www.moutaki.tech` had a broken markdown link in `server_name` so Nginx did not recognize it.

---

### Q2 — You said we can use the same domain for many websites — how?

Yes! You use **subdomains** + **multiple Nginx server blocks**.

One domain, many apps:

```
moutaki.tech          → Spring Boot backend (port 8080)
app.moutaki.tech      → React frontend (port 3000)
admin.moutaki.tech    → Admin panel (port 4000)
api.moutaki.tech      → Another API (port 5000)
```

Each subdomain gets its own `server` block in Nginx:

```nginx
server {
    listen 443 ssl;
    server_name api.moutaki.tech;
    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

server {
    listen 443 ssl;
    server_name app.moutaki.tech;
    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

In Cloudflare DNS you add each subdomain pointing to the same server IP:

| Name  | Type | Value           |
|-------|------|-----------------|
| @     | A    | 16.192.104.128  |
| www   | A    | 16.192.104.128  |
| app   | A    | 16.192.104.128  |
| api   | A    | 16.192.104.128  |

All point to the same EC2 server. Nginx decides which app to forward to based on the subdomain.

> 💡 This is exactly what you will do for your frontend — add a new subdomain like `app.moutaki.tech` pointing to the same IP, and add a new Nginx block for it.

---

### Q3 — Why do we use Cloudflare for HTTPS? Is SSL not free?

**SSL certificates from public CAs can cost money** — but there are also free options like Let's Encrypt (Certbot). However, Cloudflare gives you something even better for free:

**What Cloudflare gives you for free:**
- ✔ Public SSL certificate (browser sees the green padlock) — free
- ✔ Origin Certificate for your server — free
- ✔ DDoS protection — free
- ✔ CDN (your content loads faster globally) — free
- ✔ DNS management — free
- ✔ Firewall rules — free

**Why Cloudflare instead of Certbot?**

| | Certbot (Let's Encrypt) | Cloudflare |
|---|---|---|
| Cost | Free | Free |
| Certificate renewal | Every 90 days (auto) | 15 years (Origin Cert) |
| DDoS protection | ❌ | ✔ |
| CDN | ❌ | ✔ |
| Hides your real IP | ❌ | ✔ |
| Setup difficulty | Medium | Easy |

> 💡 Cloudflare hides your real server IP from the internet. Attackers see Cloudflare's IP, not yours. That alone is a huge security benefit.

---

### Q4 — My backend has a domain but my frontend does not — what do I do?

Since your project is separated (backend + frontend), you add a **subdomain for the frontend**.

**Example plan:**
```
moutaki.tech        → Backend (Spring Boot) — already done ✔
app.moutaki.tech    → Frontend (React/Vue/etc)
```

**Steps:**

**1. In Cloudflare DNS — add new record:**

| Name | Type | Value          | Proxy |
|------|------|----------------|-------|
| app  | A    | 16.192.104.128 | 🟠 ON |

**2. On your EC2 server — deploy frontend and add Nginx block:**

```bash
sudo nano /etc/nginx/sites-available/frontend
```

```nginx
server {
    listen 443 ssl;
    server_name app.moutaki.tech;

    ssl_certificate     /etc/ssl/cloudflare/origin.pem;
    ssl_certificate_key /etc/ssl/cloudflare/origin.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/frontend /etc/nginx/sites-enabled/frontend
sudo nginx -t
sudo systemctl restart nginx
```

Done — `https://app.moutaki.tech` will serve your frontend.

---

### Q5 — How did we get to Cloudflare nameservers? Step by step:

Here is the full journey from buying the domain to Cloudflare managing it:

**Step 1 — Buy domain on Namecheap**
- Go to namecheap.com, search `moutaki.tech`, buy it.
- Namecheap is now your **domain registrar** (they registered the domain for you).

**Step 2 — Add domain to Cloudflare**
- Go to cloudflare.com → Add a Site → enter `moutaki.tech`
- Choose the Free plan
- Cloudflare scans your existing DNS records

**Step 3 — Cloudflare gives you 2 nameservers**
```
lennojn.ns.cloudflare.com
savanna.ns.cloudflare.com
```
These are Cloudflare's own DNS servers.

**Step 4 — You go back to Namecheap and change the nameservers**
- Namecheap Dashboard → your domain → Nameservers → Custom DNS
- Enter the two Cloudflare nameservers
- Save

**Step 5 — Now Cloudflare controls your DNS**
- Cloudflare becomes your DNS provider
- Any DNS record you add (A, CNAME, etc.) is managed in Cloudflare, not Namecheap

---

### Q6 — Why do we change nameservers? What happens if we don't?

**Nameservers** are like the "phone book" of the internet. When someone types `moutaki.tech`, their browser asks the nameserver: *"Where is this domain?"*

By default, Namecheap's nameservers answer that question. But Cloudflare cannot protect, proxy, or give HTTPS to your domain if Namecheap's servers are in charge.

**When you switch nameservers to Cloudflare:**
- Cloudflare answers all DNS questions for your domain
- Cloudflare can now proxy traffic through its network
- You get DDoS protection, SSL, CDN — all of it
- Your real server IP is hidden behind Cloudflare's IP

**If you did NOT change nameservers:**
- Cloudflare cannot proxy your traffic
- No DDoS protection
- No free SSL from Cloudflare
- Your real server IP would be exposed publicly
- The orange cloud in Cloudflare DNS would do nothing

> 💡 Think of it like this: Namecheap sold you the domain name, but Cloudflare is now the one answering the phone when someone calls that name. You transferred the "receptionist" from Namecheap to Cloudflare.

---
---

# AWS EC2 + Nginx + Cloudflare HTTPS Setup Guide
> Spring Boot on Docker · Nginx Reverse Proxy · Cloudflare SSL

---

## What This Guide Does

Sets up a complete HTTPS pipeline for a Spring Boot app running on AWS EC2 (Ubuntu), behind Nginx, with Cloudflare as the DNS and SSL proxy.

**After completing these steps:**
- `http://yourdomain.com` → automatically redirects to HTTPS
- `https://yourdomain.com` → serves your Spring Boot app securely
- Cloudflare handles the public SSL certificate
- Nginx handles SSL on the server side using a Cloudflare Origin Certificate

---

## Architecture Overview

```
Browser → Cloudflare (HTTPS:443) → Your EC2 Nginx (HTTPS:443) → Spring Boot (HTTP:8080)
```

### Why did HTTPS fail before?
Cloudflare **Full SSL** mode connects to your origin server on **port 443**. Your server only had port 80 open. So Cloudflare timed out — that is the **522 error**.

---

## Prerequisites

- AWS EC2 instance running Ubuntu
- Spring Boot app running in Docker on port 8080
- Nginx installed and running on port 80
- A domain pointing to Cloudflare (orange cloud enabled)
- Cloudflare account with your domain added

---

## Step-by-Step Setup

### Step 1 — Fix the Nginx Config

Open the config file:
```bash
sudo nano /etc/nginx/sites-available/springboot
```

Make sure it contains exactly this (replace `yourdomain.com`):
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

> ⚠️ `server_name` must be plain hostnames only — no markdown links, no brackets, no extra characters.

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

Test and reload:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

Test HTTP works:
```bash
curl -v -H "Host: yourdomain.com" http://127.0.0.1/
# Should return HTTP 200
```

---

### Step 2 — Create a Cloudflare Origin Certificate

1. Go to **Cloudflare Dashboard** → your domain → **SSL/TLS** → **Origin Server** → **Create Certificate**
2. Settings:
   - Key type: `RSA`
   - Hostnames: `yourdomain.com` and `*.yourdomain.com`
   - Validity: `15 years`
3. Click **Create**

Cloudflare will give you:
- **Certificate** (starts with `-----BEGIN CERTIFICATE-----`)
- **Private Key** (starts with `-----BEGIN PRIVATE KEY-----`)

> ⚠️ Do NOT close this page. The Private Key is shown only once.

---

### Step 3 — Save the Certificate on Your Server

Create the directory:
```bash
sudo mkdir -p /etc/ssl/cloudflare
```

Save the certificate:
```bash
sudo nano /etc/ssl/cloudflare/origin.pem
```
Paste the full certificate. It must look like this:
```
-----BEGIN CERTIFICATE-----
MIIEpDCCA4ygAwIBAgIU... (long base64 text)
-----END CERTIFICATE-----
```

Save the private key:
```bash
sudo nano /etc/ssl/cloudflare/origin.key
```
Paste the full private key. It must look like this:
```
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkq... (long base64 text)
-----END PRIVATE KEY-----
```

Set secure permissions:
```bash
sudo chmod 600 /etc/ssl/cloudflare/origin.key
```

> ⚠️ Always include the `-----BEGIN-----` and `-----END-----` lines. Without them, Nginx will fail to load the certificate.

---

### Step 4 — Update Nginx to Support HTTPS

Open your Nginx config:
```bash
sudo nano /etc/nginx/sites-available/springboot
```

Replace everything with this full config:
```nginx
# Redirect all HTTP traffic to HTTPS
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$host$request_uri;
}

# Handle HTTPS with Cloudflare Origin Certificate
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/ssl/cloudflare/origin.pem;
    ssl_certificate_key /etc/ssl/cloudflare/origin.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

Test and reload:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

Verify port 443 is now listening:
```bash
sudo ss -tlnp | grep ':443'
# Should show nginx on 0.0.0.0:443
```

---

### Step 5 — Open Port 443 in AWS Security Group

1. Go to **AWS Console** → **EC2** → **Security Groups**
2. Select your instance's security group
3. Click **Edit inbound rules** → **Add rule**:
   - Type: `HTTPS`
   - Port: `443`
   - Source: `0.0.0.0/0`
4. Click **Save rules**

> ⚠️ Without this step, Cloudflare cannot reach port 443 on your server even if Nginx is listening.

---

### Step 6 — Set Cloudflare SSL to Full (Strict)

1. Go to **Cloudflare Dashboard** → your domain → **SSL/TLS** → **Overview**
2. Change SSL mode to: **Full (strict)**

> 💡 Full (strict) works because Cloudflare trusts its own Origin Certificate. This gives you end-to-end encryption.

---

## Verify Everything Works

```bash
curl -v https://yourdomain.com
```

You should see:
- Connected to yourdomain.com on port 443
- TLSv1.3 handshake successful
- SSL certificate verify ok
- HTTP 200 response from your Spring Boot app

Then open a browser and go to `https://yourdomain.com` — you should see the green padlock 🔒

---

## Troubleshooting

### Nginx config test fails
```bash
sudo nginx -t
```
Common causes:
- Wrong path to certificate files
- Missing `-----BEGIN-----` / `-----END-----` lines in `.pem` or `.key`
- Typo in `server_name` (no markdown, no brackets)

### Still getting 522 after setup
- Check AWS Security Group has port 443 open
- Check Nginx is listening: `sudo ss -tlnp | grep ':443'`
- Check Cloudflare SSL mode is **Full** or **Full (strict)**, not Flexible

### Getting 403 on HTTP
`server_name` does not match the Host header. Make sure it has both `yourdomain.com` and `www.yourdomain.com` as plain text separated by a space.

### Certificate error in Nginx
Check your `.pem` file:
```bash
cat /etc/ssl/cloudflare/origin.pem
```
- First line must be: `-----BEGIN CERTIFICATE-----`
- Last line must be: `-----END CERTIFICATE-----`

---

## Quick Reference Commands

```bash
# Check what is listening on ports
sudo ss -tlnp | grep -E ':80|:443|:8080'

# Test Nginx config
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx

# View Nginx error logs
sudo tail -50 /var/log/nginx/error.log

# View Nginx access logs
sudo tail -50 /var/log/nginx/access.log

# Check Docker containers
docker ps

# Test Spring Boot directly
curl http://localhost:8080

# Test with domain Host header
curl -v -H "Host: yourdomain.com" http://127.0.0.1/

# Test HTTPS end to end
curl -v https://yourdomain.com
```

---

*Setup completed on moutaki.tech · June 2026*
