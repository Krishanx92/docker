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

# API Portal 1.0.0 — Platform API image, on Ubuntu. See README.md in the product
# directory for how to build it.
#
# The binary uses cgo, so a different base image must be glibc-based (not Alpine).

ARG PORT=9243

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

FROM ubuntu:24.04

ARG TARGETARCH
ARG PORT
# true for the experimental (event-gateway plugin) build; recorded as a label.
ARG EXPERIMENTAL=false
ARG USERNAME=wso2
ARG USER_UID=10001
ARG USER_GID=$USER_UID

LABEL org.opencontainers.image.experimental="${EXPERIMENTAL}"

RUN apt-get update && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN groupadd -g $USER_UID appgroup && \
    useradd -u $USER_UID -g appgroup -s /bin/bash -m $USERNAME

COPY --from=binary /platform-api/bin/linux-${TARGETARCH}/platform-api ./platform-api
COPY --from=binary /platform-api/internal/database/schema.sql            ./internal/database/schema.sql
COPY --from=binary /platform-api/internal/database/schema.postgres.sql   ./internal/database/schema.postgres.sql
COPY --from=binary /platform-api/internal/database/schema.sqlserver.sql  ./internal/database/schema.sqlserver.sql
COPY --from=binary /platform-api/internal/database/schema.sqlite.sql     ./internal/database/schema.sqlite.sql
COPY --from=binary /platform-api/resources/default-llm-provider-templates ./resources/default-llm-provider-templates
COPY --from=binary /platform-api/resources/openapi.yaml                  ./resources/openapi.yaml

# Writable by the user and the root group (OpenShift runs with an arbitrary UID in GID 0).
RUN mkdir -p /app/data /app/data/certs && \
    chown -R wso2:appgroup /app && \
    chgrp -R 0 /app && \
    chmod -R g=u /app

EXPOSE ${PORT}

ENV LOG_LEVEL=INFO
ENV PORT=${PORT}

USER $USERNAME

ENTRYPOINT ["./platform-api"]
