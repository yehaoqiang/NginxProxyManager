# This is a Dockerfile intended to be built using `docker buildx`
# for multi-arch support. Building with `docker build` may have unexpected results.

# This file assumes that the frontend has been built using ./scripts/frontend-build
# and the output is available in 'frontend/dist' directory in the build context.

# --- STAGE 1: Backend Dependency Builder ---
# This stage builds the backend's node_modules for production.
FROM nginxproxymanager/nginx-full:certbot-node AS backend-deps-builder
# ARGs needed if package.json scripts use them during 'yarn install --production'
ARG BUILD_VERSION
ARG BUILD_COMMIT
ARG BUILD_DATE
ENV NPM_BUILD_VERSION="${TAG_NAME}" \
    NPM_BUILD_COMMIT="${BUILD_COMMIT}" \
    NPM_BUILD_DATE="${BUILD_DATE}" \
    NODE_ENV=production

WORKDIR /app
# Copy only necessary files for dependency installation
COPY backend/package.json backend/yarn.lock* ./
# Install only production dependencies and clean cache
RUN yarn install --production --frozen-lockfile && yarn cache clean

# --- Utility stages for CAs (as in your original Dockerfile) ---
FROM nginxproxymanager/testca AS testca
FROM letsencrypt/pebble AS pebbleca

# --- FINAL APPLICATION IMAGE ---
# Use the same base image for the final stage
FROM nginxproxymanager/nginx-full:certbot-node

# Re-declare ARGs needed in this stage
ARG TARGETPLATFORM
ARG BUILD_VERSION
ARG BUILD_COMMIT
ARG BUILD_DATE

# See: https://github.com/just-containers/s6-overlay/blob/master/README.md
ENV SUPPRESS_NO_CONFIG_WARNING=1 \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=1 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_FIX_ATTRS_HIDDEN=1 \
    S6_KILL_FINISH_MAXTIME=10000 \
    S6_VERBOSITY=1 \
    NODE_ENV=production \
    NPM_BUILD_VERSION="${BUILD_VERSION}" \
    NPM_BUILD_COMMIT="${BUILD_COMMIT}" \
    NPM_BUILD_DATE="${BUILD_DATE}" \
    NPM_LANGUAGE="zh" \
    NODE_OPTIONS="--openssl-legacy-provider"

# Install runtime packages and clean up in one layer.
# Consider if jq and logrotate are essential at runtime.
# If jq is only for a setup script (e.g., from rootfs),
# it might be better to install/uninstall it within that script's execution if possible,
# or ensure it's very small.
RUN echo "fs.file-max = 65535" > /etc/sysctl.conf \
    && apt-get update \
    && apt-get install -y --no-install-recommends jq logrotate \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# s6 overlay
COPY docker/scripts/install-s6 /tmp/install-s6
RUN /tmp/install-s6 "${TARGETPLATFORM}" && rm -f /tmp/install-s6

EXPOSE 80 81 443

# Copy pre-built backend node_modules from the builder stage
COPY --from=backend-deps-builder /app/node_modules /app/node_modules

# Copy application code - ensure .dockerignore is used effectively for these
COPY backend       /app
COPY frontend/dist /app/frontend
COPY global        /app/global

WORKDIR /app

# Add late to limit cache-busting by modifications
COPY docker/rootfs /
COPY --from=pebbleca /test/certs/pebble.minica.pem /etc/ssl/certs/pebble.minica.pem
COPY --from=testca /home/step/certs/root_ca.crt /etc/ssl/certs/NginxProxyManager.crt

COPY dnspod/client.py /tmp/dnspod_client.py
RUN set -e; \
    pip config set global.index-url https://mirrors.huaweicloud.com/repository/pypi/simple; \
    pip install --no-cache-dir certbot-dns-dnspod zope -i https://pypi.org/simple; \
    PKG_PATH=$(python -c "import os, certbot_dns_dnspod; print(os.path.dirname(certbot_dns_dnspod.__file__))" 2>/dev/null || echo "certbot_dns_dnspod_not_found"); \
    echo "certbot_dns_dnspod Path detected as: $PKG_PATH"; \
    if [ -n "$PKG_PATH" ] && [ "$PKG_PATH" != "certbot_dns_dnspod_not_found" ] && [ -d "$PKG_PATH" ]; then \
        echo "Copying patched client.py to $PKG_PATH"; \
        cp /tmp/dnspod_client.py "$PKG_PATH/client.py"; \
    else \
        echo "Warning: PKG_PATH for certbot_dns_dnspod ('$PKG_PATH') is invalid or package not found. Patched dnspod client.py NOT copied."; \
    fi; \
    rm -f /tmp/dnspod_client.py

RUN rm -rf /etc/s6-overlay/s6-rc.d/user/contents.d/frontend /etc/nginx/conf.d/dev.conf \
    && chmod 644 /etc/logrotate.d/nginx-proxy-manager

VOLUME [ "/data" ]
ENTRYPOINT [ "/init" ]

LABEL maintainer="Konveksa" \
    org.label-schema.schema-version="1.0" \
    org.label-schema.license="MIT" \
    org.label-schema.name="nginx-proxy-manager" \
    org.label-schema.description="Docker container for managing Nginx proxy hosts with a simple, powerful interface " \
    org.label-schema.url="https://github.com/yehaoqiang/NginxProxyManager" \
    org.label-schema.vcs-url="https://github.com/yehaoqiang/NginxProxyManager.git" \