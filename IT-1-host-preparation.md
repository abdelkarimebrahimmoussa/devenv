# IT-1 — Host Preparation: Swap, Cleanup, Firewall

**Server:** 46.224.122.127  
**SSH Port:** 22369  
**User:** root  
**Date Completed:** 2026-03-10  
**Status:** ✅ PASSED  
**Depends On:** Nothing — first iteration  
**Next:** IT-2 — Nginx Reverse Proxy + SSL  

---

## What This Iteration Does

Prepares the bare server before any containers or services are created:
1. Creates a swap file so OOM killer does not silently kill containers under memory pressure
2. Cleans up any stale Docker resources (dangling images, stopped containers, unused volumes)
3. Installs and configures UFW firewall — blocks everything inbound except SSH, HTTP, HTTPS

---

## Execution

Run all commands as `root` directly on the server over SSH.

### IT-1.1 — Create Swap File

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap sw 0 0" >> /etc/fstab
```

> **Note:** This server is 4GB RAM — swap sized to 4GB.  
> Production server (Hetzner 251GB RAM) should use `fallocate -l 16G /swapfile` instead.

---

### IT-1.2 — Disk Cleanup

```bash
docker system prune -f
docker volume prune -f
```

> **Note:** Skip if Docker is not yet installed. Run `docker --version` first to check.  
> This server had Docker not installed at time of execution — step was skipped.

---

### IT-1.3 — Install and Configure UFW Firewall

```bash
apt update -y
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow 22369/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

> **IMPORTANT:** Outbound is intentionally unrestricted. AI agents inside containers  
> must reach Anthropic API, GitHub, Google AI, Deepseek, and other external URLs freely.

---

## How to Test (Run After Execution)

Run each command and verify the expected output before moving to IT-2.

### Test 1 — Swap is active
```bash
free -h | grep Swap
```
**Expected:** `Swap:` line shows value greater than 0 (e.g. `4.0Gi` or `16Gi`)  
**Fail if:** Shows `0B`

---

### Test 2 — Disk space available
```bash
df -h /
```
**Expected:** `Avail` column shows significant free space (was 31GB on this server)  
**Fail if:** Less than 10GB free

---

### Test 3 — UFW is active with correct rules
```bash
ufw status
```
**Expected output:**
```
Status: active

To                         Action      From
--                         ------      ----
22369/tcp                  ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```
**Fail if:** Status is inactive, or any of the three ports are missing

---

### Test 4 — SSH still connects after firewall is enabled
Run from your **local machine** (not the server):
```bash
ssh -p 22369 root@46.224.122.127
```
**Expected:** Connects successfully  
**Fail if:** Connection refused or timeout — means UFW blocked SSH, you need console access to fix

---

### Test 5 — Port 80 is blocked (nothing listening yet)
Run from your **local machine**:
```bash
curl --max-time 5 http://46.224.122.127
```
**Expected:** Connection refused or empty response (no service on 80 yet — correct)  
**Fail if:** Times out without response AND UFW is not active (would mean firewall itself is the issue)

---

## Results on This Server

| Test | Command | Result |
|------|---------|--------|
| 1 — Swap active | `free -h` | `Swap: 4.0Gi` ✅ |
| 2 — Disk free | `df -h /` | 31GB available ✅ |
| 3 — UFW rules | `ufw status` | active, 22369/80/443 ALLOW ✅ |
| 4 — SSH connects | `ssh -p 22369` | Connected ✅ |
| 5 — Port 80 no service | `curl http://46.224.122.127` | No response (correct) ✅ |
