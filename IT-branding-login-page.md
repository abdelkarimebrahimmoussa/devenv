# IT-Branding — NYOTA Login Page Redesign
Server: 46.224.122.127 | Date: 2026-03-10 | Status: COMPLETE | Depends-on: IT-5 | Next: IT-6

## What This Iteration Does
Replaces the default code-server login page with a fully branded NYOTA login page.
Left panel: full-bleed desert/landscape photo with brand tagline overlay.
Right panel: centered NYOTA SVG logo mark, wordmark, gold-accented form on near-black surface.
The branding is baked into nyota-base:latest — all role containers inherit it automatically.

## Execution

```bash
# 1. Place branded login.html in base image
mkdir -p /opt/nyota-studio/base/branding
# (HTML file with embedded base64 image transferred via SCP)

# 2. Rebuild base image (Dockerfile already has COPY branding/login.html)
cd /opt/nyota-studio/base
docker build -t nyota-base:latest .

# 3. Rebuild role image (inherits from nyota-base)
cd /opt/nyota-studio
docker build -t nyota-flutter:latest ./roles/flutter/

# 4. Restart container
docker stop nyota-flutter1 && docker rm nyota-flutter1
docker run -d   --name nyota-flutter1   --env-file /opt/nyota-studio/secrets/flutter.env   -v nyota-flutter1-workspace:/workspace   -v /data/claude-home/.claude:/home/coder/.claude   -p 127.0.0.1:8801:8080   --restart unless-stopped   nyota-flutter:latest
```

## How to Test

```bash
# 1. Confirm container running
docker ps | grep nyota-flutter1
# Expected: Up status

# 2. Confirm branded page served
curl -s http://127.0.0.1:8801/login | grep -o 'NYOTA\|Agent Development'
# Expected: NYOTA, NYOTA, Agent Development

# 3. Open in browser
# https://flutter1.dev.nyota.studio → should show dark NYOTA login, not default code-server
```

## Results on This Server

| Test | Command | Expected | Result |
|------|---------|----------|--------|
| Container up | docker ps \| grep nyota-flutter1 | Up status | ✅ PASS |
| Branding in page | curl localhost:8801/login \| grep NYOTA | NYOTA found | ✅ PASS |
| Label updated | curl localhost:8801/login \| grep Agent | Agent Development found | ✅ PASS |
| Browser render | https://flutter1.dev.nyota.studio | Dark NYOTA login | ✅ PASS |

## Design Decisions
- Near-black surface: #111111 — matches NYOTA brand identity
- Gold accent: #C9A84C — used on role tag and input focus ring
- Cream: #FFFDF8 — all text and submit button
- Photo: embedded as base64 JPEG (73KB compressed from 14MB original)
- CSP-compliant: no external fonts or CDN — all self-contained
- Role badge: shows {{WELCOME_TEXT}} (container role name from code-server)
