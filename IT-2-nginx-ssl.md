# IT-2 — Nginx Reverse Proxy + SSL

**Server:** 46.224.122.127  
**SSH Port:** 22369  
**User:** root  
**Date Completed:** 2026-03-10  
**Status:** ✅ PASSED  
**Depends On:** IT-1 (UFW firewall must be active)  
**Next:** IT-3 — Cloudflare Zero Trust via Tunnel  

---

## What This Iteration Does

Installs Nginx as the single HTTPS entry point for all container subdomains.
Every role container gets its own subdomain mapped to a local port.
SSL wildcard certificate covers `*.dev.nyota.studio` via Let's Encrypt (manual DNS challenge).
HTTP on port 80 redirects to HTTPS. Certbot auto-renewal hook reloads Nginx after each renewal.

> **Architecture note:** DNS records for `*.dev.nyota.studio` use grey cloud (DNS only, no Cloudflare proxy).
> This is intentional — our Let's Encrypt cert handles SSL directly. Cloudflare Zero Trust auth
> is implemented via Cloudflare Tunnel in IT-3 (more secure than proxy for internal tooling).
> Production domains on `nyota.studio` are completely unaffected.

---

## Execution

### IT-2.1 — Install Nginx and Certbot

```bash
apt update -y
apt install nginx certbot python3-certbot-nginx -y
systemctl enable nginx
systemctl start nginx
```

---

### IT-2.2 — Point DNS to Server (Cloudflare)

In Cloudflare DNS for `nyota.studio`, add two records — **grey cloud (DNS only)**:

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| A | `*.dev` | `46.224.122.127` | Grey (DNS only) |
| A | `dev` | `46.224.122.127` | Grey (DNS only) |

> **Important:** Must be grey cloud, NOT orange. Orange cloud causes TLS handshake failure
> because Cloudflare's free Universal SSL does not cover `*.dev.nyota.studio` (two levels deep).

---

### IT-2.3 — Obtain Wildcard SSL Certificate

Run interactively — certbot will prompt for two DNS TXT records:

```bash
certbot certonly --manual --preferred-challenges dns \
  -d "*.dev.nyota.studio" -d "dev.nyota.studio" \
  --agree-tos --email admin@nyota.studio --force-interactive
```

When certbot shows each TXT value, add it in Cloudflare:

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| TXT | `_acme-challenge.dev` | (value shown by certbot) | Grey (DNS only) |

> Certbot requests **two TXT records** — keep both, add the second without removing the first.
> Verify propagation before pressing Enter: `nslookup -type=TXT _acme-challenge.dev.nyota.studio 1.1.1.1`

Certificate saved at: `/etc/letsencrypt/live/dev.nyota.studio/`  
Expires: **2026-06-08** (89 days)

---

### IT-2.4 — Nginx Config (All Containers Pre-Configured)

Config written to `/etc/nginx/sites-available/dev-studio`, symlinked to sites-enabled.
Includes HTTP→HTTPS redirect and server blocks for all 21 subdomains.

Port mapping:

| Subdomain | Host Port |
|-----------|-----------|
| flutter1.dev.nyota.studio | 8801 |
| flutter2.dev.nyota.studio | 8802 |
| flutter3.dev.nyota.studio | 8803 |
| arch1.dev.nyota.studio | 8810 |
| arch2.dev.nyota.studio | 8811 |
| plan1.dev.nyota.studio | 8820 |
| plan2.dev.nyota.studio | 8821 |
| back1.dev.nyota.studio | 8830 |
| back2.dev.nyota.studio | 8831 |
| back3.dev.nyota.studio | 8832 |
| exec1.dev.nyota.studio | 8840 |
| exec2.dev.nyota.studio | 8841 |
| exec3.dev.nyota.studio | 8842 |
| exec4.dev.nyota.studio | 8843 |
| exec5.dev.nyota.studio | 8844 |
| review.dev.nyota.studio | 8850 |
| ux1.dev.nyota.studio | 8860 |
| ux2.dev.nyota.studio | 8861 |
| devops.dev.nyota.studio | 8870 |
| litellm.dev.nyota.studio | 4000 |
| cchv.dev.nyota.studio | 8900 |

Reload after any config change:
```bash
nginx -t && nginx -s reload
```

---

### IT-2.5 — Certbot Auto-Renewal Hook

```bash
mkdir -p /etc/letsencrypt/renewal-hooks/post
printf "#!/bin/bash\nsystemctl reload nginx\n" > /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

> **Note:** The wildcard cert was obtained with `--manual`, so certbot cannot auto-renew it
> without an auth hook script. Renewal requires re-running IT-2.3 before 2026-06-08.
> Set a calendar reminder for 2026-05-25 to renew.

---

## How to Test

### Test 1 — Nginx config syntax valid
```bash
nginx -t
```
**Expected:** `syntax is ok` and `test is successful`  
**Fail if:** Any error

---

### Test 2 — Nginx is running
```bash
systemctl is-active nginx
```
**Expected:** `active`  
**Fail if:** `inactive` — run `systemctl start nginx`

---

### Test 3 — HTTP redirects to HTTPS
```bash
curl -sI http://flutter1.dev.nyota.studio | head -2
```
**Expected:** `HTTP/1.1 301 Moved Permanently`  
**Fail if:** 200 or connection refused

---

### Test 4 — HTTPS serving with valid cert (502 = correct, no container yet)
```bash
curl -sI https://flutter1.dev.nyota.studio | head -2
```
**Expected:** `HTTP/1.1 502 Bad Gateway` with no SSL errors  
**Fail if:** SSL handshake error or connection refused

---

### Test 5 — Real cert is installed and valid
```bash
certbot certificates
```
**Expected:** `*.dev.nyota.studio` listed, valid expiry date shown  
**Fail if:** No certificates found

---

### Test 6 — Certbot timer active
```bash
systemctl is-active certbot.timer
```
**Expected:** `active`  
**Fail if:** `inactive`

---

## Results on This Server

| Test | Command | Result |
|------|---------|--------|
| 1 — Nginx syntax | `nginx -t` | syntax ok ✅ |
| 2 — Nginx active | `systemctl is-active nginx` | active ✅ |
| 3 — HTTP → 301 | `curl -sI http://flutter1.dev.nyota.studio` | 301 Moved Permanently ✅ |
| 4 — HTTPS 502 valid SSL | `curl -sI https://flutter1.dev.nyota.studio` | 502 Bad Gateway ✅ |
| 5 — Real cert installed | `certbot certificates` | valid until 2026-06-08 ✅ |
| 6 — certbot.timer active | `systemctl is-active certbot.timer` | active ✅ |

---

## ⚠️ SSL Renewal Reminder

Manual wildcard cert expires **2026-06-08**. Re-run IT-2.3 before that date.  
Set reminder for **2026-05-25**.
