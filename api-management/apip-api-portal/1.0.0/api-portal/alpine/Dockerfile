# --------------------------------------------------------------------
# Copyright (c) 2026, WSO2 LLC. (https://www.wso2.com).
#
# WSO2 LLC. licenses this file to you under the Apache License,
# Version 2.0 (the "License"); you may not use this file except
# in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
# --------------------------------------------------------------------

# API Portal 1.0.0 — API Portal image, on Alpine. See README.md in the product
# directory for how to build it.
#
# When changing the base image, keep the image's runtime contract: working
# directory /app, UID 10001, writable /app/data and /app/logs, port 9543, base
# config at $APIP_AP_CONFIG_PATH, started with `node src/server.js`.

# The product version to build is passed with
# --build-arg PRODUCT_VERSION=1.0.0.<update level>.
ARG BINARY_REPOSITORY=registry.wso2.com/wso2-api-platform/binaries/api-portal
ARG PRODUCT_VERSION

# Downloads the product binary. Your registry credentials are used only here, and
# never reach the image.
FROM --platform=$BUILDPLATFORM ghcr.io/oras-project/oras:v1.3.6@sha256:9cfb578be975b762fd898b9d004243d8003aeb807a3e15b2ed212cdd1326c9eb AS fetch
ARG BINARY_REPOSITORY
ARG PRODUCT_VERSION
RUN --mount=type=secret,id=REGISTRY_USERNAME \
    --mount=type=secret,id=REGISTRY_PASSWORD \
    set -eu; \
    if [ -z "${PRODUCT_VERSION:-}" ]; then \
        echo "Error: pass the product version, e.g. --build-arg PRODUCT_VERSION=1.0.0.7" >&2; \
        exit 1; \
    fi; \
    if [ ! -s /run/secrets/REGISTRY_USERNAME ] || [ ! -s /run/secrets/REGISTRY_PASSWORD ]; then \
        echo "Error: pass your registry credentials, e.g. --secret id=REGISTRY_USERNAME --secret id=REGISTRY_PASSWORD" >&2; \
        exit 1; \
    fi; \
    oras login "${BINARY_REPOSITORY%%/*}" -u "$(cat /run/secrets/REGISTRY_USERNAME)" \
        --password-stdin < /run/secrets/REGISTRY_PASSWORD; \
    oras pull "${BINARY_REPOSITORY}:${PRODUCT_VERSION}" -o /tmp/pull; \
    set -- /tmp/pull/*.zip; \
    if [ "$#" -ne 1 ] || [ ! -f "$1" ]; then \
        echo "Error: expected one zip in ${BINARY_REPOSITORY}:${PRODUCT_VERSION}" >&2; \
        exit 1; \
    fi; \
    mkdir /binary; \
    unzip -q "$1" -d /binary; \
    rm -rf /tmp/pull "${HOME}/.docker"

FROM scratch AS binary
COPY --from=fetch /binary/ /

FROM node:24-alpine AS build
# Root for the setup steps; some base images default to a non-root user.
USER 0
WORKDIR /app
COPY --from=binary /api-portal/package.json /api-portal/package-lock.json /api-portal/.npmrc ./
RUN npm ci --omit=dev

FROM node:24-alpine
USER 0
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=binary /api-portal/package.json /api-portal/package-lock.json ./
COPY --from=binary /api-portal/src ./src
COPY --from=binary /api-portal/database ./database
COPY --from=binary /api-portal/resources ./resources

# npm and npx are not used at runtime.
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx

# Writable by UID 10001 and the root group (OpenShift runs with an arbitrary UID in GID 0).
RUN mkdir -p /app/data /app/logs \
 && chown -R 10001:0 /app/data /app/logs \
 && chmod -R g=u /app/data /app/logs

# Base config file; `--config <path>` arguments are layered on top of it.
ENV APIP_AP_CONFIG_PATH=/app/configs/config.toml

EXPOSE 9543
USER 10001:10001
ENTRYPOINT ["node", "src/server.js"]
