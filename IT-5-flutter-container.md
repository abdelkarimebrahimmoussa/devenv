# IT-5 — First Role Container: Flutter Dev

**Server:** 46.224.122.127  
**SSH Port:** 22369  
**User:** root  
**Date Completed:** 2026-03-10  
**Status:** ✅ PASSED  
**Depends On:** IT-4 (nyota-base image)  
**Next:** IT-6 — Deploy All Remaining Role Containers  

---

## What This Iteration Does

Builds the Flutter Dev role container on top of `nyota-base:latest`.
Adds Flutter 3.x SDK, the Dart/Flutter VS Code extension, and runs it on
port 8801 behind the Nginx block already configured in IT-2.
This is the pilot container — all other role containers follow the same pattern.

---

## Execution

### IT-5.1 — Create Flutter Dockerfile

```bash
cat > /opt/nyota-studio/roles/flutter/Dockerfile << 'DOCKER'
FROM nyota-base:latest

# Flutter SDK
RUN git clone https://github.com/flutter/flutter.git \
    -b stable /opt/flutter --depth 1

ENV PATH="/opt/flutter/bin:$PATH"

RUN flutter doctor --android-licenses -y 2>/dev/null || true
RUN flutter precache --no-android --no-ios --no-web --no-macos --no-linux --no-windows --no-fuchsia || flutter precache || true

# Flutter VS Code extension
RUN code-server --install-extension dart-code.flutter 2>/dev/null || true

LABEL nyota.role="flutter-dev"

CMD ["code-server", "--bind-addr", "0.0.0.0:8080", "--auth", "password", "/workspace"]
DOCKER
```

### IT-5.2 — Create Secrets File

```bash
cat > /opt/nyota-studio/secrets/flutter.env << 'ENV'
ANTHROPIC_API_KEY=your_key_here
GOOGLE_AI_API_KEY=your_key_here
DEEPSEEK_API_KEY=your_key_here
PASSWORD=nyota2026dev
ENV
chmod 600 /opt/nyota-studio/secrets/flutter.env
```

### IT-5.3 — Create Workspace Volume

```bash
docker volume create nyota-flutter1-workspace
```

### IT-5.4 — Build Image

```bash
docker build -t nyota-flutter:latest /opt/nyota-studio/roles/flutter/
# ~3-4 minutes (Flutter SDK download ~218MB)
```

### IT-5.5 — Run Container

```bash
docker run -d \
  --name nyota-flutter1 \
  --env-file /opt/nyota-studio/secrets/flutter.env \
  -v nyota-flutter1-workspace:/workspace \
  -v /data/claude-home/.claude:/home/coder/.claude \
  -p 127.0.0.1:8801:8080 \
  --restart unless-stopped \
  nyota-flutter:latest
```

> Nginx block for flutter1.dev.nyota.studio → 127.0.0.1:8801 was already
> created in IT-2. No Nginx changes needed.

---

## How to Test

### Test 1 — Container running
```bash
docker ps | grep nyota-flutter1
```
**Expected:** `Up` status

### Test 2 — code-server started
```bash
docker logs nyota-flutter1 2>&1 | head -5
```
**Expected:** i18next line (code-server initializing)

### Test 3 — HTML on local port
```bash
curl -s --max-time 5 http://127.0.0.1:8801/login | grep -c 'html'
```
**Expected:** Number > 0

### Test 4 — HTTPS via Nginx returns valid response
```bash
curl -sI --max-time 8 https://flutter1.dev.nyota.studio
```
**Expected:** `HTTP/1.1 302 Found` (code-server login redirect over HTTPS)

### Test 5 — Browser (manual)
Open https://flutter1.dev.nyota.studio → enter password from PASSWORD env var → VS Code loads

### Test 6 — Flutter version
```bash
docker exec nyota-flutter1 flutter --version
```
**Expected:** `Flutter 3.x.x • channel stable`

### Test 7 — Claude Code version
```bash
docker exec nyota-flutter1 claude --version
```
**Expected:** `x.x.x (Claude Code)`

### Test 8 — API key injected
```bash
docker exec nyota-flutter1 printenv ANTHROPIC_API_KEY
```
**Expected:** The key value (confirming .env injection)

### Test 9 — nyota-login sets git identity
```bash
echo -e 'Dev User\ndev@nyota.studio\nfaketoken' | docker exec -i nyota-flutter1 bash -c 'nyota-login && git config --global user.name'
```
**Expected:** Last line: `Dev User`

### Test 10 — Dirty repo blocks logout
```bash
docker exec nyota-flutter1 bash -c 'mkdir -p /workspace/testrepo && cd /workspace/testrepo && git init -q && touch dirty.txt && nyota-logout 2>&1; echo exit:$?'
```
**Expected:** Warning shown, `exit:1`

### Test 11 — Clean logout works
```bash
docker exec nyota-flutter1 bash -c 'rm -rf /workspace/testrepo && nyota-logout'
```
**Expected:** `✅ Logged out. Git credentials cleared.`

### Test 12 — Workspace volume persists across restart
```bash
docker exec nyota-flutter1 bash -c 'echo persist_test > /workspace/volume_check.txt'
docker restart nyota-flutter1
sleep 6
docker exec nyota-flutter1 cat /workspace/volume_check.txt
```
**Expected:** `persist_test`

---

## Results on This Server

| Test | Result |
|------|--------|
| 1 — Container Up | Up ✅ |
| 2 — code-server logs | i18next startup line ✅ |
| 3 — HTML on 8801 | 3 matches ✅ |
| 4 — HTTPS 302 via Nginx | 302 Found ✅ |
| 5 — Browser (manual) | VS Code opens ✅ |
| 6 — flutter --version | Flutter 3.41.4 stable ✅ |
| 7 — claude --version | 2.1.72 (Claude Code) ✅ |
| 8 — API key injected | Value printed ✅ |
| 9 — nyota-login | Returns "Dev User" ✅ |
| 10 — Dirty blocks logout | exit:1 ✅ |
| 11 — Clean logout | Credentials cleared ✅ |
| 12 — Volume persists | "persist_test" after restart ✅ |

---

## Notes

- Flutter version: 3.41.4 channel stable
- Image size: ~3.5GB uncompressed (Flutter SDK is large)
- Password set in `/opt/nyota-studio/secrets/flutter.env` → `PASSWORD=` field
- The `ANTHROPIC_API_KEY=your_key_here` placeholder must be replaced with real keys before team use
- Shared CCHV volume mounted at `/data/claude-home/.claude` (prepared for IT-9)
- Build time: ~3-4 minutes (Flutter SDK download dominates)
- Subsequent builds are faster due to layer caching
