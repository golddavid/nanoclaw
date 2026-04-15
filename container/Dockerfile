# NanoClaw Agent Container
# Runs Claude Agent SDK in isolated Linux VM with browser automation

FROM ubuntu:24.04

# Suppress interactive prompts during apt installs
ENV DEBIAN_FRONTEND=noninteractive

# Install Node.js 22 via NodeSource
RUN apt-get update && apt-get install -y ca-certificates curl gnupg \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Install system dependencies for Chromium
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    fonts-noto-cjk \
    fonts-noto-color-emoji \
    libgbm1 \
    libnss3 \
    libatk-bridge2.0-0 \
    libgtk-3-0 \
    libx11-xcb1 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libasound2t64 \
    libpangocairo-1.0-0 \
    libcups2 \
    libdrm2 \
    libxshmfence1 \
    curl \
    git \
    jq \
    && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" > /etc/apt/sources.list.d/github-cli.list \
    && apt-get update && apt-get install -y gh \
    && rm -rf /var/lib/apt/lists/*

# Install rtk (token-optimized CLI proxy) — arch-aware
RUN ARCH=$(dpkg --print-architecture) && \
    if [ "$ARCH" = "amd64" ]; then \
        curl -fsSL https://github.com/rtk-ai/rtk/releases/download/v0.36.0/rtk_amd64.deb -o /tmp/rtk.deb \
        && dpkg -i /tmp/rtk.deb && rm /tmp/rtk.deb; \
    elif [ "$ARCH" = "arm64" ]; then \
        curl -fsSL https://github.com/rtk-ai/rtk/releases/download/v0.36.0/rtk-aarch64-unknown-linux-gnu.tar.gz -o /tmp/rtk.tar.gz \
        && tar -xzf /tmp/rtk.tar.gz -C /usr/local/bin --strip-components=0 \
        && chmod +x /usr/local/bin/rtk && rm /tmp/rtk.tar.gz \
        && (rtk --version >/dev/null 2>&1 || (echo "RTK ARM64 binary requires newer glibc than this image provides — removing (hook degrades gracefully)" && rm -f /usr/local/bin/rtk)); \
    fi

# Configure git to use gh as credential helper (token injected at runtime via GH_TOKEN)
RUN git config --global credential.helper '!gh auth git-credential' \
    && git config --global user.email "nanoclaw-agent@localhost" \
    && git config --global user.name "NanoClaw Agent"

# Set Chromium path for agent-browser
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium

# Install agent-browser and claude-code globally
RUN npm install -g agent-browser @anthropic-ai/claude-code

# Create app directory
WORKDIR /app

# Copy package files first for better caching
COPY agent-runner/package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY agent-runner/ ./

# Build TypeScript
RUN npm run build

# Rename the default ubuntu user/group (uid/gid 1000) to node
RUN usermod -l node ubuntu && usermod -d /home/node -m node && groupmod -n node ubuntu

# Create workspace directories
RUN mkdir -p /workspace/group /workspace/global /workspace/extra /workspace/ipc/messages /workspace/ipc/tasks /workspace/ipc/input

# Create entrypoint script
# Container input (prompt, group info) is passed via stdin JSON.
# Credentials are injected by the host's credential proxy — never passed here.
# Follow-up messages arrive via IPC files in /workspace/ipc/input/
RUN printf '#!/bin/bash\nset -e\ncd /app && npx tsc --outDir /tmp/dist 2>&1 >&2\nln -s /app/node_modules /tmp/dist/node_modules\nchmod -R a-w /tmp/dist\ncat > /tmp/input.json\nnode /tmp/dist/index.js < /tmp/input.json\n' > /app/entrypoint.sh && chmod +x /app/entrypoint.sh

# Install rtk Claude Code hook for the node user
COPY hooks/rtk-rewrite.sh /home/node/.claude/hooks/rtk-rewrite.sh
RUN chmod +x /home/node/.claude/hooks/rtk-rewrite.sh \
    && printf '{"hooks":{"PreToolUse":[{"matcher":"Bash","hooks":[{"type":"command","command":"/home/node/.claude/hooks/rtk-rewrite.sh"}]}]}}\n' \
       > /home/node/.claude/settings.json

# Set ownership to node user (non-root) for writable directories
RUN chown -R node:node /workspace /home/node/.claude && chmod 777 /home/node

# Switch to non-root user (required for --dangerously-skip-permissions)
USER node

# Set working directory to group workspace
WORKDIR /workspace/group

# Entry point reads JSON from stdin, outputs JSON to stdout
ENTRYPOINT ["/app/entrypoint.sh"]
