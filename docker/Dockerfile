# This is a Dockerfile intended to be built using `docker buildx`
# for multi-arch support. Building with `docker build` may have unexpected results.

# This file assumes that the frontend has been built using ./scripts/frontend-build

FROM nginxproxymanager/testca AS testca
FROM letsencrypt/pebble AS pebbleca
FROM nginxproxymanager/nginx-full:certbot-node

ARG TARGETPLATFORM
ARG BUILD_VERSION
ARG BUILD_COMMIT
ARG BUILD_DATE

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

RUN echo "fs.file-max = 65535" > /etc/sysctl.conf \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends jq logrotate \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/lists/*

# s6 overlay
COPY docker/scripts/install-s6 /tmp/install-s6
RUN /tmp/install-s6 "${TARGETPLATFORM}" && rm -f /tmp/install-s6

EXPOSE 80 81 443

COPY backend       /app
COPY frontend/dist /app/frontend
COPY global        /app/global

WORKDIR /app
RUN yarn install \
	&& yarn cache clean

COPY docker/rootfs /
COPY --from=pebbleca /test/certs/pebble.minica.pem /etc/ssl/certs/pebble.minica.pem
COPY --from=testca /home/step/certs/root_ca.crt /etc/ssl/certs/NginxProxyManager.crt

COPY dnspod/client.py /tmp/client.py

RUN rm -rf /etc/s6-overlay/s6-rc.d/user/contents.d/frontend /etc/nginx/conf.d/dev.conf \
	&& chmod 644 /etc/logrotate.d/nginx-proxy-manager

RUN pip config set global.index-url https://mirrors.huaweicloud.com/repository/pypi/simple && \
	pip install --no-cache-dir certbot-dns-dnspod && \
	pip install --no-cache-dir zope

RUN PKG_PATH=$(python -c "import os, certbot_dns_dnspod; print(os.path.dirname(certbot_dns_dnspod.__file__))") && \
    echo "Certbot DNSPod Path is: $PKG_PATH" && \
    mkdir -p "$PKG_PATH" && \
    cp /tmp/client.py "$PKG_PATH/client.py" && \
    rm /tmp/client.py

VOLUME [ "/data" ]
ENTRYPOINT [ "/init" ]

LABEL maintainer="Konveksa" \
   	org.label-schema.schema-version="1.0" \
	org.label-schema.license="MIT" \
	org.label-schema.name="nginx-proxy-manager" \
	org.label-schema.description="Docker container for managing Nginx proxy hosts with a simple, powerful interface " \
	org.label-schema.url="https://github.com/jc21/nginx-proxy-manager" \
	org.label-schema.vcs-url="https://github.com/jc21/nginx-proxy-manager.git" \
	org.label-schema.cmd="docker run --rm -ti jc21/nginx-proxy-manager:latest"
