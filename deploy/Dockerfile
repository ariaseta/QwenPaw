# syntax=docker/dockerfile:1.7
# -----------------------------------------------------------------------------
# Optimized for BuildKit:
#   • Cache mounts for apt / npm / pip → near-instant rebuilds
#   • Layered COPY (lockfile → install → source) → maximum cache hits
#   • Single combined apt layer → fewer layers, smaller image
# -----------------------------------------------------------------------------

ARG NODE_IMAGE=agentscope-registry.ap-southeast-1.cr.aliyuncs.com/agentscope/node:slim

# -----------------------------------------------------------------------------
# Stage 1: build console frontend (dist not committed in repo).
# -----------------------------------------------------------------------------
FROM ${NODE_IMAGE} AS console-builder
WORKDIR /app/console

# Increase Node.js heap size for large vite builds.
ENV NODE_OPTIONS="--max-old-space-size=4096"

# Console build uses git metadata in npm scripts.
RUN apt-get update && apt-get install -y --no-install-recommends git

# 1. Install deps first — cached unless package*.json changes.
COPY console/package.json console/package-lock.json ./
RUN --mount=type=cache,target=/root/.npm,sharing=locked \
    npm ci --include=dev --prefer-offline --no-audit --no-fund

# 2. Copy source and build.
COPY console/ ./
RUN npm run build

# -----------------------------------------------------------------------------
# Stage 2: runtime image with Python, Chromium, and app.
# -----------------------------------------------------------------------------
FROM ${NODE_IMAGE} AS runtime

ENV NODE_ENV=production \
    WORKSPACE_DIR=/app \
    QWENPAW_WORKING_DIR=/app/working \
    QWENPAW_SECRET_DIR=/app/working.secret \
    QWENPAW_BACKUP_DIR=/app/working.backups \
    DEBIAN_FRONTEND=noninteractive \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_INPUT=1

# Channel filtering: use QWENPAW_DISABLED_CHANNELS (exclusion, recommended)
# or QWENPAW_ENABLED_CHANNELS (whitelist). Override at runtime with -e.
ARG QWENPAW_DISABLED_CHANNELS="imessage"
ARG QWENPAW_ENABLED_CHANNELS=""
ENV QWENPAW_DISABLED_CHANNELS=${QWENPAW_DISABLED_CHANNELS} \
    QWENPAW_ENABLED_CHANNELS=${QWENPAW_ENABLED_CHANNELS}

# ── System packages (single layer, with apt cache mount) ─────────────────────
# Disable apt auto-clean so cache mount can persist between builds.
RUN rm -f /etc/apt/apt.conf.d/docker-clean \
 && echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
      curl \
      python3 \
      python3-pip \
      python3-venv \
      build-essential \
      libssl-dev \
      git \
      supervisor \
      vim \
      gettext-base \
      xfce4 \
      xfce4-terminal \
      xvfb \
      dbus-x11 \
      fonts-wqy-zenhei \
      fonts-wqy-microhei \
      chromium \
      chromium-sandbox \
      libx11-xcb1 \
      libxcomposite1 \
      libxdamage1 \
      libxext6 \
      libxfixes3 \
      libxi6 \
      libxtst6 \
      libnss3 \
      libglib2.0-0 \
      libdrm2 \
      libgbm1 \
      libasound2 \
      fonts-liberation \
      libu2f-udev

# Run chromium with --no-sandbox in container.
RUN sed -i 's/^CHROMIUM_FLAGS=""/CHROMIUM_FLAGS="--no-sandbox"/' /usr/bin/chromium

# Playwright: use system Chromium (already installed above).
ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium \
    PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    QWENPAW_RUNNING_IN_CONTAINER=1

WORKDIR ${WORKSPACE_DIR}

# ── Python venv ──────────────────────────────────────────────────────────────
RUN python3 -m venv venv
ENV PATH="/app/venv/bin:$PATH"

# ── Python deps + app install (with pip cache mount) ─────────────────────────
# Copying pyproject/setup first lets the next install benefit from cache when
# only src/ changes (pip will still re-link, but wheel downloads stay cached).
COPY pyproject.toml setup.py README.md ./
COPY src ./src
COPY --from=console-builder /app/console/dist/ ./src/qwenpaw/console/

RUN --mount=type=cache,target=/root/.cache/pip,sharing=locked \
    pip install .

# Init working dir with default config.json and HEARTBEAT.md (build time).
RUN qwenpaw init --defaults --accept-security

# QwenPaw app port (default 8088). Override at runtime with -e QWENPAW_PORT=3000.
ENV QWENPAW_PORT=8088

COPY deploy/config/supervisord.conf.template /etc/supervisor/conf.d/supervisord.conf.template
COPY --chmod=755 deploy/entrypoint.sh /entrypoint.sh

EXPOSE 8088

CMD ["/entrypoint.sh"]
