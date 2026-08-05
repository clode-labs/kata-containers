# Copyright (c) 2021 Intel Corporation
#
# SPDX-License-Identifier: Apache-2.0

FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive
ENV INSTALL_IN_GOPATH=false
# Required for libxml2-dev
ENV TZ=Etc/UTC
ARG ARCH

COPY install_yq.sh /usr/bin/install_yq.sh
COPY install_oras.sh /usr/bin/install_oras.sh

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# clode: apt on GitHub runners intermittently fails to reach the Ubuntu
# mirrors. Every asset build starts by building this image, so a single
# failed fetch takes out a whole build stage — we hit it three times across
# four runs, on build-firecracker and build-rootfs-image alike. It surfaces
# as a bare "exit code: 100" with no apt output, because
# kata-deploy-binaries-in-docker.sh invokes `docker build -q`.
#
# apt does not retry by default. Give it retries and a longer timeout.
RUN printf 'Acquire::Retries "5";\nAcquire::Retries::Delay::Maximum "30";\nAcquire::http::Timeout "60";\nAcquire::https::Timeout "60";\n' \
    > /etc/apt/apt.conf.d/99clode-retries

# Install yq, oras, and docker
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        docker.io \
        curl \
        make \
        git \
        wget \
        sudo && \
    apt-get clean && rm -rf /var/lib/apt/lists/ && \
    install_yq.sh && \
    install_oras.sh

ARG IMG_USER=kata-builder
ARG UID=1000
ARG GID=1000
# gid of the docker group on the host, required for running docker in docker builds.
ARG HOST_DOCKER_GID

RUN if [ ${IMG_USER} != "root" ]; then sed -i -e "/:${GID}:/d" /etc/group; groupadd --gid=${GID} ${IMG_USER};fi
RUN if [ ${IMG_USER} != "root" ]; then adduser ${IMG_USER} --uid=${UID} --gid=${GID};fi
RUN if [ ${IMG_USER} != "root" ] && [ ! -z ${HOST_DOCKER_GID} ] && [ ! $(getent group | grep :${HOST_DOCKER_GID}:) ]; then groupadd --gid=${HOST_DOCKER_GID} docker_on_host;fi
RUN if [ ${IMG_USER} != "root" ] && [ ! -z ${HOST_DOCKER_GID} ]; then usermod -a -G ${HOST_DOCKER_GID} ${IMG_USER};fi
RUN sh -c "echo '${IMG_USER} ALL=NOPASSWD: ALL' >> /etc/sudoers"

RUN if [ "${ARCH}" != "$(uname -m)" ] && [ "${ARCH}" == "s390x" ]; then sed -i 's/^deb/deb [arch=amd64]/g' /etc/apt/sources.list && \
    dpkg --add-architecture "s390x" && \
    echo "deb [arch=s390x] http://ports.ubuntu.com/ jammy main multiverse universe" >> /etc/apt/sources.list && \
    echo "deb [arch=s390x] http://ports.ubuntu.com/ jammy-security main multiverse universe" >> /etc/apt/sources.list && \
    echo "deb [arch=s390x] http://ports.ubuntu.com/ jammy-backports main multiverse universe" >> /etc/apt/sources.list && \
    echo "deb [arch=s390x] http://ports.ubuntu.com/ jammy-updates main multiverse universe" >> /etc/apt/sources.list; fi

#FIXME: gcc is required as agent is build out of a container build.
# clode: retry update+install as a unit. archive.ubuntu.com is load balanced,
# and its backends drift out of sync during an archive rotation: the index one
# host serves can reference a .deb another host has already dropped, giving a
# hard 404 mid-install. Acquire::Retries does not help — a 404 is not transient
# from apt's point of view. Re-running apt-get update re-resolves the index,
# usually against a backend that agrees with the pool.
RUN apt_ok=0; \
    for attempt in 1 2 3 4 5; do \
      if apt-get update && \
         apt-get install --no-install-recommends -y \
         build-essential \
         cpio \
         gcc \
         unzip \
         git \
         make \
         wget \
         xz-utils \
         zstd; then apt_ok=1; break; fi; \
      echo "clode: apt attempt ${attempt}/5 failed, retrying in 20s" >&2; \
      rm -rf /var/lib/apt/lists/*; \
      sleep 20; \
    done; \
    [ "${apt_ok}" = 1 ] || { echo "clode: apt failed after 5 attempts" >&2; exit 1; }; \
    if [ "${ARCH}" != "$(uname -m)" ] && [ "${ARCH}" == "s390x" ]; then \
        apt-get install -y --no-install-recommends \
        gcc-s390x-linux-gnu \
        g++-s390x-linux-gnu \
        binutils-s390x-linux-gnu \
        dpkg-dev \
        apt-utils \
        libssl-dev:s390x \
        libcurl4-openssl-dev:s390x \
        libjson-c-dev:s390x \
        pkg-config:s390x \
        libxml2-dev:s390x \
        libjson-c-dev:s390x \
        libglib2.0-0:s390x \
        libglib2.0-dev:s390x; \
    elif uname -m | grep -Eq 's390x'; then apt-get install -y s390-tools; fi && \
    apt-get clean && rm -rf /var/lib/apt/lists

RUN if [ "${ARCH}" != "$(uname -m)" ] && [ "${ARCH}" == "s390x" ]; then \
        git clone -b v2.25.0 https://github.com/ibm-s390-linux/s390-tools.git && cd s390-tools && \
        pushd genprotimg && pushd boot && make CROSS_COMPILE=s390x-linux-gnu- && popd && pushd src && \
        make CROSS_COMPILE=s390x-linux-gnu- && popd && make install && popd || return; fi

ENV USER ${IMG_USER}
USER ${IMG_USER}
