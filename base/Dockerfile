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

# NYOTA branded login page
COPY branding/login.html /usr/lib/code-server/src/browser/pages/login.html

WORKDIR /workspace

EXPOSE 8080

CMD ["code-server", "--bind-addr", "0.0.0.0:8080", "--auth", "password", "/workspace"]
