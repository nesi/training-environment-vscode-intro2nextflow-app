FROM ghcr.io/linuxserver/baseimage-ubuntu:jammy

# set version label
ARG BUILD_DATE
ARG VERSION
ARG CODE_RELEASE
ARG NXF_VER=25.10.2
ARG NF_CORE_DEMO_VER=1.1.0
LABEL build_version="Linuxserver.io version:- ${VERSION} Build-date:- ${BUILD_DATE}"
LABEL maintainer="Jen Reeve"

# environment settings
ARG DEBIAN_FRONTEND="noninteractive"
ENV HOME="/config"

RUN \
  echo "**** install runtime dependencies ****" && \
  apt-get update && \
  apt-get install -y \
    curl \
    dnsutils \
    git \
    jq \
    ldap-utils \
    libnss-ldapd \
    libpam-ldapd \
    libatomic1 \
    nano \
    net-tools \
    netcat \
    nodejs \
    nscd \
    nslcd \
    rsync \
    unzip \
    vim \
    wget \
    zip \
    openjdk-17-jdk openjdk-17-jre \
    python3 python3-pip \
    sudo && \
  echo "**** install code-server ****" && \
  if [ -z ${CODE_RELEASE+x} ]; then \
    CODE_RELEASE=$(curl -sX GET https://api.github.com/repos/coder/code-server/releases/latest \
      | awk '/tag_name/{print $4;exit}' FS='[""]' | sed 's|^v||'); \
  fi && \
  mkdir -p /app/code-server && \
  curl -o \
    /tmp/code-server.tar.gz -L \
    "https://github.com/coder/code-server/releases/download/v${CODE_RELEASE}/code-server-${CODE_RELEASE}-linux-amd64.tar.gz" && \
  tar xf /tmp/code-server.tar.gz -C \
    /app/code-server --strip-components=1 && \
  echo "**** clean up ****" && \
  apt-get clean && \
  rm -rf \
    /config/* \
    /tmp/* \
    /var/lib/apt/lists/* \
    /var/tmp/* 


# install add-apt-repository and friends, then add the PPA
RUN apt-get update \
    && apt-get install -y --no-install-recommends software-properties-common gnupg ca-certificates \
    && add-apt-repository -y ppa:apptainer/ppa \
    && apt-get update \
    && apt-get install -y --no-install-recommends apptainer-suid \
    && rm -rf /var/lib/apt/lists/*s

# install kubectl, required for running on the k8s cluster
ARG KUBECTL_VERSION=v1.28.5
RUN curl -LO https://dl.k8s.io/release/$KUBECTL_VERSION/bin/linux/amd64/kubectl \
    && mv kubectl /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

# copy in ood k8s utils
ARG UTILS_HASH=6298fb01f7a7c66a8454e3f0fd74437a32491423
RUN git clone https://github.com/nesi/training-environment-k8s-utils.git /opt/ood-k8s-utils \
    && cd /opt/ood-k8s-utils \
    && git checkout $UTILS_HASH \
    && chmod +x /opt/ood-k8s-utils/files/* \
    && mv /opt/ood-k8s-utils/files/* /bin/ \
    && rm -rf /opt/ood-k8s-utils

# make a dummy module command to avoid warnings from ondemand job_script_content.sh
RUN echo "#!/bin/bash" > /bin/module \
    && chmod +x /bin/module

RUN set -eux; \
    curl -fsSL https://get.nextflow.io | bash; \
    mv nextflow /usr/local/bin/nextflow; \
    chmod 755 /usr/local/bin/nextflow; \
    nextflow self-update "${NXF_VER}"

RUN nextflow -version

# install nf-core
RUN python3 -m pip install --no-cache-dir --upgrade pip setuptools wheel \
    && python3 -m pip install --no-cache-dir nf-core

RUN nf-core --version

# RUN nf-core pipelines download demo -r ${NF_CORE_DEMO_VER} --outdir /var/lib/nf-core-demo \
#   --container-system singularity --compress none

RUN export NXF_HOME=/var/lib/.nextflow \
  && export NXF_APPTAINER_CACHEDIR=/var/lib/.nextflow/apptainer \
  && nextflow run nextflow-io/hello \
  && NXF_VER=22.10.0 nextflow run nextflow-io/hello -r v1.1 \
  && NXF_VER=26.04.0 nextflow run nf-core/demo -profile test,apptainer -r ${NF_CORE_DEMO_VER} --outdir results -preview

RUN mkdir -p /var/lib/.apptainer_cache && cd /var/lib/.apptainer_cache \
  && apptainer pull docker://community.wave.seqera.io/library/multiqc:1.33--ee7739d47738383b \
  && apptainer pull docker://quay.io/biocontainers/seqtk:1.4--he4a0461_1 \
  && apptainer pull docker://quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0 

RUN ln -s /app/code-server/bin/code-server /usr/local/bin/code-server

# ports and volumes
EXPOSE 8443