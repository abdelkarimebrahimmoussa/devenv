# IT-3 — Cloudflare Zero Trust Authentication

**Server:** 46.224.122.127  
**SSH Port:** 22369  
**User:** root  
**Date Completed:** 2026-03-10  
**Status:** ✅ PASSED  
**Depends On:** IT-1 (UFW), IT-2 (Nginx + SSL)  
**Next:** IT-4 — Base Docker Image  

---

## What This Iteration Does

Secures all `*.dev.nyota.studio` subdomains so only authorized team members can access them.
DNS records point directly to the server (grey cloud, DNS only — no Cloudflare proxy).
Our Let's Encrypt wildcard cert handles SSL end-to-end.
Security is enforced at the application layer via code-server password per container.
Cloudflare Tunnel was evaluated and rejected (see Architecture Decisions below).

---

## Architecture Decision — Why Grey Cloud, Not Tunnel

During this iteration, Cloudflare Tunnel was attempted as the auth layer.
It was abandoned for the following reasons:

| Option | Result | Reason |
|--------|--------|--------|
| Orange cloud + Cloudflare proxy | ❌ | Free Universal SSL doesn't cover `*.dev.nyota.studio` (two levels deep) |
| Cloudflare Tunnel + Zero Trust Access | ❌ | Zero Trust Access requires paid plan ($7/user/month) |
| Grey cloud + Let's Encrypt cert | ✅ | Direct DNS, our cert handles SSL, clean and working |

The tunnel CNAME records were created and then deleted. The final state is grey cloud A records.

**Security model without Zero Trust:**
- UFW firewall blocks all direct inbound except 80/443/22369
- HTTPS enforced — HTTP redirects to HTTPS
- code-server password per container (set in `.env` per role)
- Server IP is technically visible in DNS — acceptable for an internal dev team

**Future upgrade path:** If Zero Trust Access is needed later, upgrade to Cloudflare Zero Trust paid plan and re-enable the tunnel. The cloudflared binary is already installed on the server.

---

## Execution

### IT-3.1 — Install cloudflared (already done, kept for re-runs)

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | gpg --dearmor -o /usr/share/keyrings/cloudflare-main.gpg
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main' > /etc/apt/sources.list.d/cloudflared.list
apt update -y
apt install cloudflared -y
cloudflared --version
```

### IT-3.2 — DNS Records (Cloudflare Dashboard)

In Cloudflare → nyota.studio → DNS → Records, ensure these two records exist:

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| A | `*.dev` | `46.224.122.127` | 🔘 Grey cloud (DNS only) |
| A | `dev` | `46.224.122.127` | 🔘 Grey cloud (DNS only) |

> **Critical:** Must be grey cloud. Orange cloud causes TLS handshake failure
> because Cloudflare's free Universal SSL doesn't cover `*.dev.nyota.studio`.

### IT-3.3 — Verify cloudflared is stopped

```bash
systemctl stop cloudflared
systemctl disable cloudflared
systemctl is-active cloudflared   # must return: inactive
```

---

## How to Test

### Test 1 — DNS resolves to our server IP
```bash
nslookup flutter1.dev.nyota.studio 1.1.1.1
```
**Expected:** `Address: 46.224.122.127`  
**Fail if:** Returns Cloudflare IPs (172.67.x.x or 104.21.x.x) — means orange cloud is ON

---

### Test 2 — HTTPS returns 502 (correct — no container yet)
```bash
curl -sI --max-time 10 https://flutter1.dev.nyota.studio
```
**Expected:** `HTTP/1.1 502 Bad Gateway` from nginx — valid SSL, no cert errors  
**Fail if:** SSL handshake error — check grey cloud is set in DNS

---

### Test 3 — HTTP redirects to HTTPS
```bash
curl -sI --max-time 10 http://flutter1.dev.nyota.studio
```
**Expected:** `HTTP/1.1 301 Moved Permanently`  
**Fail if:** Connection refused or 200

---

### Test 4 — cloudflared is not running
```bash
systemctl is-active cloudflared
```
**Expected:** `inactive`  
**Fail if:** `active` — run `systemctl stop cloudflared && systemctl disable cloudflared`

---

## Results on This Server

| Test | Command | Result |
|------|---------|--------|
| 1 — DNS → our IP | `nslookup flutter1.dev.nyota.studio 1.1.1.1` | 46.224.122.127 ✅ |
| 2 — HTTPS 502 | `curl -sI https://flutter1.dev.nyota.studio` | 502 Bad Gateway ✅ |
| 3 — HTTP → 301 | `curl -sI http://flutter1.dev.nyota.studio` | 301 Moved Permanently ✅ |
| 4 — tunnel stopped | `systemctl is-active cloudflared` | inactive ✅ |

---

## Notes

- cloudflared binary is installed at `/usr/bin/cloudflared` — available for future use
- Tunnel `nyota-dev-studio` exists in Cloudflare account (ID: c8911862-8a1d-4549-baa2-e3726240a54c)
- Two TXT records for `_acme-challenge.dev` still exist in DNS — safe to delete after cert renewal
- SSL cert expires 2026-06-08 — renew before that date (see IT-2)
