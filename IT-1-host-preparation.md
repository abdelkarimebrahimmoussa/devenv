# IT-1 — Host Preparation: Swap, Cleanup, Firewall

**Server:** 46.224.122.127 | **Date:** 2026-03-10 | **Status:** ✅ PASSED

---

## Commands Executed (All Passed)

### IT-1.1 — Create Swap File

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap sw 0 0" >> /etc/fstab
```

> Note: Test server is 4GB RAM — swap sized to 4GB. Production (251GB RAM) should use 16GB swap.

### IT-1.2 — Disk Cleanup

```bash
# Docker not installed yet — prune skipped
df -h /
# Result: 31GB available
```

### IT-1.3 — Install and Configure UFW

```bash
apt update -y
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow 22369/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
ufw status
```

---

## Verification Results

| # | Test | Result |
|---|------|--------|
| 1 | Swap active | Swap: 4.0Gi ✅ |
| 2 | Disk space | 31GB available ✅ |
| 3 | UFW active + 22369/80/443 rules | active ✅ |
| 4 | SSH still connects on 22369 | Connected ✅ |
| 5 | Port 80 — no service (correct) | Connection refused ✅ |

---

## Notes

- UFW outbound unrestricted — AI agents can reach Anthropic, GitHub, Google AI, Deepseek
- /swapfile in /etc/fstab — survives reboots
- Docker not yet installed (comes in IT-4)

**Next:** IT-2 — Nginx Reverse Proxy + SSL
