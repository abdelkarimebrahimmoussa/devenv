# IT-2 — Nginx Reverse Proxy + SSL

**Server:** 46.224.122.127  
**SSH Port:** 22369  
**User:** root  
**Date Completed:** 2026-03-10  
**Status:** ✅ PASSED (SSL pending DNS — see note)  
**Depends On:** IT-1 (UFW firewall must be active)  
**Next:** IT-3 — Cloudflare Zero Trust Authentication  

---

## What This Iteration Does

Installs Nginx as the single HTTPS entry point for all container subdomains.
Every role container gets its own subdomain mapped to a local port.
SSL certificate covers `*.dev.nyota.studio` via Let's Encrypt wildcard.
HTTP on port 80 redirects to HTTPS. Certbot auto-renewal hook reloads Nginx after each renewal.

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

### IT-2.2 — Point DNS Wildcard to Server

In Cloudflare DNS, create these two records (orange cloud proxy ON):

| Type | Name | Value |
|------|------|-------|
| A | `*.dev.nyota.studio` | `46.224.122.127` |
| A | `dev.nyota.studio` | `46.224.122.127` |

Wait 2–5 minutes for propagation before running certbot.

> **Note:** This is a manual step in Cloudflare dashboard — cannot be automated here.

---

### IT-2.3 — Obtain Wildcard SSL Certificate

Run interactively — certbot will ask you to add a DNS TXT record:

```bash
certbot certonly --manual --preferred-challenges dns \
  -d "*.dev.nyota.studio" -d "dev.nyota.studio"
```

When prompted, add the TXT record in Cloudflare DNS, wait 30 seconds, then press Enter to confirm.

> **Note:** Dummy self-signed cert created at `/etc/letsencrypt/live/dev.nyota.studio/` during testing.
> Replace with real cert after DNS is configured. Real cert path will be the same.

---

### IT-2.4 — Nginx Config (All Containers Pre-Configured)

Config file already written to `/etc/nginx/sites-available/dev-studio` and symlinked to sites-enabled.
All 18 container server blocks are defined. Port mapping:

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

To reload after any config change:
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

---

## How to Test

### Test 1 — Nginx config syntax valid
```bash
nginx -t
```
**Expected:** `syntax is ok` and `test is successful`  
**Fail if:** Any error — fix config before proceeding

---

### Test 2 — Nginx is running
```bash
systemctl is-active nginx
```
**Expected:** `active`  
**Fail if:** `inactive` or `failed` — run `systemctl start nginx`

---

### Test 3 — HTTP redirects to HTTPS
```bash
curl -sI http://localhost | head -3
```
**Expected:** `HTTP/1.1 301 Moved Permanently`  
**Fail if:** Returns 200 or connection refused

---

### Test 4 — HTTPS is serving (502 expected — no containers yet)
```bash
curl -sk https://localhost | head -c 200
```
**Expected:** `502 Bad Gateway` (Nginx reached, no container behind it yet — correct)  
**Fail if:** SSL error or connection refused — means cert or config issue

---

### Test 5 — Certbot timer active (auto-renewal)
```bash
systemctl status certbot.timer | grep Active
```
**Expected:** `active (waiting)`  
**Fail if:** inactive — run `systemctl enable --now certbot.timer`

---

### Test 6 — After real SSL cert (run after IT-2.3 with real DNS)
```bash
certbot certificates
```
**Expected:** Certificate listed for `*.dev.nyota.studio` with valid future expiry  
**Fail if:** No certificates found

---

### Test 7 — After real SSL cert from external (run from local machine)
```bash
curl -I https://flutter1.dev.nyota.studio
```
**Expected:** `502 Bad Gateway` with valid SSL (no certificate warning)  
**Fail if:** SSL error or connection refused

---

## Results on This Server

| Test | Command | Result |
|------|---------|--------|
| 1 — Nginx syntax | `nginx -t` | syntax ok ✅ |
| 2 — Nginx active | `systemctl is-active nginx` | active ✅ |
| 3 — HTTP → 301 | `curl -sI http://localhost` | 301 Moved Permanently ✅ |
| 4 — HTTPS 502 | `curl -sk https://localhost` | 502 Bad Gateway ✅ |
| 5 — certbot.timer | `systemctl status certbot.timer` | active (waiting) ✅ |
| 6 — Real SSL cert | Pending DNS setup | ⏳ Needs DNS first |
| 7 — External HTTPS | Pending DNS setup | ⏳ Needs DNS first |

---

## ⚠️ DNS Action Required Before IT-3

Before running IT-3 (Cloudflare Zero Trust), the real SSL cert must be obtained.

Steps:
1. In Cloudflare DNS — add `A *.dev.nyota.studio → 46.224.122.127` and `A dev.nyota.studio → 46.224.122.127`
2. Run: `certbot certonly --manual --preferred-challenges dns -d "*.dev.nyota.studio" -d "dev.nyota.studio"`
3. Add the TXT record Certbot shows, wait 30s, confirm
4. Run: `nginx -s reload`
5. Test: `certbot certificates` — must show valid cert
