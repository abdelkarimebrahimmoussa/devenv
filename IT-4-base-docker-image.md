# IT-4 — Base Docker Image

**Server:** 46.224.122.127  
**SSH Port:** 22369  
**User:** root  
**Date Completed:** 2026-03-10  
**Status:** ✅ PASSED  
**Depends On:** IT-1 (UFW), IT-2 (Nginx), IT-3 (DNS)  
**Next:** IT-5 — First Role Container (Flutter Dev)  

---

## What This Iteration Does

Installs Docker on the host server, then builds `nyota-base:latest` — the parent image
that all role containers inherit from. The image contains: Ubuntu 24.04, core dev tools,
code-server (VS Code in browser), Claude Code CLI, and the nyota-login/logout scripts.
Every role container will start with `FROM nyota-base:latest`.

---

## Execution

### IT-4.0 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
systemctl enable docker
systemctl start docker
docker --version   # must show version
```

### IT-4.1 — Create Directory Structure

```bash
mkdir -p /opt/nyota-studio/base/scripts
mkdir -p /opt/nyota-studio/roles/{flutter,backend,architect,executor,reviewer,ux,devops,planner}
mkdir -p /opt/nyota-studio/secrets
mkdir -p /opt/nyota-studio/litellm
mkdir -p /opt/nyota-studio/nginx
mkdir -p /data/claude-home/.claude/projects
chmod 777 /data/claude-home/.claude
chmod 777 /data/claude-home/.claude/projects
```

### IT-4.2 — Create nyota-login.sh

```bash
cat > /opt/nyota-studio/base/scripts/nyota-login.sh << 'SCRIPT'
#!/bin/bash
echo "=================================="
echo "       NYOTA Dev Studio"
echo "=================================="
echo ""
read -p "Your full name: " GIT_NAME
read -p "Your email: " GIT_EMAIL
read -s -p "GitHub PAT token: " GIT_TOKEN
echo ""

git config --global user.name "$GIT_NAME"
git config --global user.email "$GIT_EMAIL"
git config --global credential.helper store
echo "https://$GIT_NAME:$GIT_TOKEN@github.com" > ~/.git-credentials
chmod 600 ~/.git-credentials

echo ""
echo "✅ Git identity set for $GIT_NAME <$GIT_EMAIL>"
echo "   Run: git config --global user.name  (to verify)"
SCRIPT
chmod +x /opt/nyota-studio/base/scripts/nyota-login.sh
```

### IT-4.3 — Create nyota-logout.sh

```bash
cat > /opt/nyota-studio/base/scripts/nyota-logout.sh << 'SCRIPT'
#!/bin/bash
echo "=================================="
echo "    NYOTA Dev Studio — Logout"
echo "=================================="
echo ""

DIRTY=false

for dir in /workspace/*/; do
  if [ -d "$dir/.git" ]; then
    STATUS=$(git -C "$dir" status --porcelain)
    if [ -n "$STATUS" ]; then
      echo "⚠️  Uncommitted changes in: $dir"
      git -C "$dir" status --short
      DIRTY=true
    fi
  fi
done

if [ "$DIRTY" = true ]; then
  echo ""
  echo "❌ CANNOT LOGOUT: Please commit or stash your changes first."
  echo "   Use: git stash   or   git commit -m \"your message\""
  exit 1
fi

rm -f ~/.git-credentials
git config --global --unset user.name 2>/dev/null
git config --global --unset user.email 2>/dev/null

echo "✅ Logged out. Git credentials cleared."
SCRIPT
chmod +x /opt/nyota-studio/base/scripts/nyota-logout.sh
```

### IT-4.4 — Create Dockerfile

```bash
cat > /opt/nyota-studio/base/Dockerfile << 'DOCKER'
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Core tools
RUN apt-get update && apt-get install -y \
    curl git wget unzip build-essential \
    python3 python3-pip nodejs npm \
    openssh-client ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# code-server (VS Code in browser)
RUN curl -fsSL https://code-server.dev/install.sh | sh

# Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code

# nyota scripts
COPY scripts/nyota-login.sh /usr/local/bin/nyota-login
COPY scripts/nyota-logout.sh /usr/local/bin/nyota-logout
RUN chmod +x /usr/local/bin/nyota-login /usr/local/bin/nyota-logout

WORKDIR /workspace

EXPOSE 8080

CMD ["code-server", "--bind-addr", "0.0.0.0:8080", "--auth", "password", "/workspace"]
DOCKER
```

### IT-4.5 — Build the Image

```bash
cd /opt/nyota-studio/base
docker build -t nyota-base:latest .
# Takes ~2 minutes on first build
```

---

## How to Test

### Test 1 — Image exists
```bash
docker images | grep nyota-base
```
**Expected:** Row showing `nyota-base` and `latest`  
**Fail if:** No output

---

### Test 2 — code-server version
```bash
docker run --rm nyota-base:latest code-server --version
```
**Expected:** Version number like `4.x.x`  
**Fail if:** Command not found

---

### Test 3 — Claude Code version
```bash
docker run --rm nyota-base:latest claude --version
```
**Expected:** `x.x.x (Claude Code)`  
**Fail if:** Command not found

---

### Test 4 — Scripts installed
```bash
docker run --rm nyota-base:latest ls -la /usr/local/bin/nyota-login /usr/local/bin/nyota-logout
```
**Expected:** Both files listed with `-rwxr-xr-x`  
**Fail if:** No such file

---

### Test 5 — nyota-login sets git identity
```bash
echo -e 'Test User\ntest@nyota.studio\nfaketoken' | docker run --rm -i nyota-base:latest bash -c 'nyota-login && git config --global user.name'
```
**Expected:** Last line prints `Test User`  
**Fail if:** No output or error

---

### Test 6 — nyota-logout clean (no repos)
```bash
docker run --rm nyota-base:latest bash -c 'nyota-logout'
```
**Expected:** `✅ Logged out. Git credentials cleared.`  
**Fail if:** Error or wrong exit code

---

### Test 7 — nyota-logout blocked by dirty repo
```bash
docker run --rm nyota-base:latest bash -c '
  mkdir -p /workspace/testrepo && cd /workspace/testrepo && git init -q && touch dirty.txt
  nyota-logout; echo exit_code:$?
'
```
**Expected:** Shows `⚠️ Uncommitted changes` warning and `exit_code:1`  
**Fail if:** `exit_code:0` (should be blocked)

---

## Results on This Server

| Test | Command | Result |
|------|---------|--------|
| 1 — Image exists | `docker images \| grep nyota-base` | nyota-base:latest 659MB ✅ |
| 2 — code-server | `code-server --version` inside container | 4.110.0 ✅ |
| 3 — Claude Code | `claude --version` inside container | 2.1.72 (Claude Code) ✅ |
| 4 — Scripts installed | `ls /usr/local/bin/nyota-*` | Both executable ✅ |
| 5 — Login sets git | nyota-login + git config | Returns "Test User" ✅ |
| 6 — Logout clean | nyota-logout (no repos) | Credentials cleared ✅ |
| 7 — Logout dirty blocked | nyota-logout with uncommitted file | exit_code:1 ✅ |

---

## Notes

- Image size: ~659MB compressed, ~2.61GB uncompressed
- code-server version: 4.110.0 with Code 1.110.0
- Claude Code version: 2.1.72
- Docker version on host: 29.3.0
- Build time: ~2 minutes (packages cached on rebuild)
