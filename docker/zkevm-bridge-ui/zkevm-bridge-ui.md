# zkevm-bridge-ui Dockerfile Documentation

This Dockerfile details the steps to build a customized Docker image for the `zkevm-bridge-ui`, tailored with specific patches and configurations for deployment.

## Overview

The build process is divided into two stages:

1. **Builder Stage**:
   - This stage is responsible for cloning the repository, applying necessary patches, and preparing the build environment.
2. **Production Stage**:
   - This stage builds the final Docker image, incorporating only the necessary artifacts from the builder stage to keep the image lightweight.

### Builder Stage

The builder stage starts from an Alpine Linux image due to its minimal size and security benefits.

```Dockerfile
FROM alpine:3.19 AS builder


STEP 1: Setup Work Environment and Dependencies
Working Directory: Sets /opt/zkevm-bridge-ui as the working directory.
Install Git and Patch: Installs necessary tools for cloning the repository and applying patches. Note the use of --no-cache to avoid storing the apk index in the final image, which is a recommended practice to keep Docker images small.

ARG ZKEVM_BRIDGE_UI_TAG
WORKDIR /opt/zkevm-bridge-ui
RUN apk add --no-cache git patch \
    && rm -rf /var/cache/apk/* \
    && git clone --branch ${ZKEVM_BRIDGE_UI_TAG} https://github.com/0xPolygonHermez/zkevm-bridge-ui .


STEP 2: Apply Patches
Patch Application: Applies predefined patches to the cloned repository, modifying scripts and configurations as necessary for the specific deployment requirements.

COPY deploy.sh.diff env.ts.diff ./
RUN patch -p1 -i deploy.sh.diff \
    && patch -p1 -i env.ts.diff


Production Stage
The production stage uses an Nginx base image optimized for running web applications.

FROM nginx:alpine
LABEL author="devtools@polygon.technology"
LABEL description="Enhanced zkevm-bridge-ui image with relative URLs support enabled"

Configuration and Final Setup
Install Node.js and NPM: Necessary for building the frontend application from the source code.
Prepare Application: Copies the package.json and related files from the builder stage, installs dependencies, and then copies the prepared application files.

RUN apk add --no-cache nodejs npm \
    && rm -rf /var/cache/apk/*

WORKDIR /app
COPY --from=builder /opt/zkevm-bridge-ui/package.json /opt/zkevm-bridge-ui/package-lock.json ./
COPY --from=builder /opt/zkevm-bridge-ui/scripts ./scripts
COPY --from=builder /opt/zkevm-bridge-ui/abis ./abis
RUN npm install
COPY --from=builder /opt/zkevm-bridge-ui/ .

WORKDIR /
ENTRYPOINT ["/bin/sh", "/app/scripts/deploy.sh"]


This Dockerfile ensures that the zkevm-bridge-ui is deployed with a configuration that supports the specific needs of the docker environment.



