# syntax=docker/dockerfile:1
# Visual Studio Code "Dev Container" Debian base image
# with common utilities needed for container development
#
# Author.........: Matteo Kutufa <mk@bytehawks.org>
# Organization...: ByteHawks <bytehawks-org>
# Base Image.....: Debian 13 (Trixie)
# Repo URL.......: https://github.com/bytehawks-org/vscode-dev-containers
FROM debian:trixie

# Default arguments for the dev container user and Node.js version
ARG VS_USER_NAME=vscode
ARG VS_USER_UID=1000
ARG VS_USER_GID=1000
ARG VS_USER_HOME=/home/$VS_USER_NAME
ARG NVM_VERSION=0.40.8
ARG NODE_VERSION=24

# Build-time metadata for OCI labels below. Left as "unknown" for local/manual builds;
# CI passes real values via --build-arg (see .github/workflows/build-debian-base.yml).
ARG IMAGE_VERSION=13
ARG IMAGE_REVISION=unknown
ARG IMAGE_CREATED=unknown

# OCI image spec annotations: https://github.com/opencontainers/image-spec/blob/main/annotations.md
LABEL org.opencontainers.image.title="Debian Base" \
      org.opencontainers.image.description="VS Code Dev Container base image for Debian 13 (Trixie) with Node.js installed via NVM" \
      org.opencontainers.image.authors="Matteo Kutufa <mk@bytehawks.org>" \
      org.opencontainers.image.vendor="ByteHawks" \
      org.opencontainers.image.licenses="Apache-2.0" \
      org.opencontainers.image.url="https://github.com/bytehawks-org/vscode-dev-containers" \
      org.opencontainers.image.source="https://github.com/bytehawks-org/vscode-dev-containers" \
      org.opencontainers.image.documentation="https://github.com/bytehawks-org/vscode-dev-containers/blob/main/README.md" \
      org.opencontainers.image.base.name="debian:trixie" \
      org.opencontainers.image.version="$IMAGE_VERSION" \
      org.opencontainers.image.revision="$IMAGE_REVISION" \
      org.opencontainers.image.created="$IMAGE_CREATED"

ENV DEBIAN_FRONTEND=noninteractive

# Replace /bin/sh with /bin/bash, install common base utilities needed to fetch and
# register the Docker apt repo, and set up locales.
# - procps: gives VS Code Server (and extensions) `ps`, required for it to detect/manage its own processes
# - locales: avoids Perl/locale warnings that otherwise surface in every shell/tool downstream
# - tini: minimal init used as ENTRYPOINT below, so PID 1 reaps zombies and forwards signals correctly
RUN rm /bin/sh \
    && ln -s /bin/bash /bin/sh \
    && apt-get update \
    && apt-get install -y --no-install-recommends curl wget git sudo \
       ca-certificates procps locales tini \
    && echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen \
    && locale-gen

# Register the Docker apt repo. The GPG key must exist on disk *before* the sources
# file that references it via Signed-By, and both must exist before the apt-get
# update below - otherwise apt fails to read the repo and docker-ce is never found.
RUN mkdir -p /etc/apt/keyrings \
    && chmod 0755 /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

RUN apt-get update \
    && apt-get install -y --no-install-recommends docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin \
    && apt-get autoclean \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8

# DEV Container default user creation and configuration
RUN groupadd --gid $VS_USER_GID $VS_USER_NAME \
    && useradd --uid $VS_USER_UID --gid $VS_USER_GID --create-home --home-dir $VS_USER_HOME $VS_USER_NAME \
    && mkdir -p $VS_USER_HOME/.local/bin \
    && chown -R $VS_USER_NAME:$VS_USER_NAME $VS_USER_HOME \
    && echo "$VS_USER_NAME ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/$VS_USER_NAME \
    && chmod 0440 /etc/sudoers.d/$VS_USER_NAME

USER $VS_USER_NAME
WORKDIR $VS_USER_HOME

ENV NVM_DIR=$VS_USER_HOME/.nvm

# Install NVM, install Node.js through it, and make it default.
# Using `nvm install` (not passing the version to install.sh, which ignores it)
# is what actually installs Node - the previous version never did.
RUN curl -sL "https://raw.githubusercontent.com/nvm-sh/nvm/refs/tags/v${NVM_VERSION}/install.sh" | bash \
    && \. "$NVM_DIR/nvm.sh" \
    && nvm install "$NODE_VERSION" \
    && nvm alias default "$NODE_VERSION" \
    && nvm cache clear \
    && ln -s "$NVM_DIR/versions/node/$(nvm version default)" "$NVM_DIR/current"

# Make nvm's shell function available in interactive shells...
RUN echo '. "$NVM_DIR/nvm.sh"' >> $VS_USER_HOME/.bashrc

# ...and put node/npm/npx permanently on PATH for every shell (login, non-login,
# non-interactive `docker exec`, VS Code tasks, etc.), independent of .bashrc being sourced.
ENV PATH=$VS_USER_HOME/.local/bin:$NVM_DIR/current/bin:$PATH

RUN node -v && npm -v && \. "$NVM_DIR/nvm.sh" && nvm --version

# tini as PID 1: reaps zombie processes and forwards signals (SIGTERM/SIGINT) correctly,
# which matters for long-lived dev containers attached to by VS Code / docker/podman exec.
# CMD is a sane default for standalone `run`; devcontainer.json's `overrideCommand` (default
# behavior) and any image built FROM this one are free to replace it without touching ENTRYPOINT.
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["sleep", "infinity"]


