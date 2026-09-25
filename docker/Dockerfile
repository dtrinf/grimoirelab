
# FROM python:3.12-slim-bookworm
FROM dhi.io/python:3.14-debian-dev
LABEL maintainer="David Trigo <david.trigo@gmail.com>"

ARG GRIMOIRELAB_RELEASE="1.22.1"

ENV DEBIAN_FRONTEND=noninteractive
ENV DEPLOY_USER=grimoire
ENV DEPLOY_USER_DIR=/home/${DEPLOY_USER}
ENV CONF_DIR=${DEPLOY_USER_DIR}/conf
ENV SCRIPTS_DIR=${DEPLOY_USER_DIR}/scripts
ENV GRIMOIRELAB_RELEASE=${GRIMOIRELAB_RELEASE}

# install dependencies
RUN apt-get update && \
    apt-get -y install --no-install-recommends \
    passwd \
    bash locales-all \
    gcc g++ gfortran \
    git git-core \
    mariadb-client \
    libmariadbclient-dev-compat \
    pkg-config \
    unzip curl wget ssh \
    && \
    apt-get clean && \
    find /var/lib/apt/lists -type f -delete

RUN apt-get update && \
    apt-get -y install --no-install-recommends \
    libopenblas-dev liblapack-dev && \
    apt-get clean && \
    find /var/lib/apt/lists -type f -delete

# Initial user
RUN useradd ${DEPLOY_USER} --create-home --shell /bin/bash

# RUN echo "${DEPLOY_USER} ALL=NOPASSWD: ALL" >> /etc/sudoers

RUN mkdir -p ${DEPLOY_USER_DIR}/logs ; chown -R ${DEPLOY_USER}:${DEPLOY_USER} ${DEPLOY_USER_DIR}/logs
VOLUME ["${DEPLOY_USER_DIR}/logs"]

RUN echo 'LANG="en_US.UTF-8"'>/etc/default/locale

ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

# Download basic configuration files
ADD https://raw.githubusercontent.com/chaoss/grimoirelab-sirmordred/main/aliases.json ${DEPLOY_USER_DIR}/aliases.json
RUN chmod 444 ${DEPLOY_USER_DIR}/aliases.json
ADD https://raw.githubusercontent.com/chaoss/grimoirelab-sirmordred/main/menu.yaml ${DEPLOY_USER_DIR}/menu.yaml
RUN chmod 444 ${DEPLOY_USER_DIR}/menu.yaml

ADD stage ${DEPLOY_USER_DIR}/stage
RUN chmod 755 ${DEPLOY_USER_DIR}/stage

# OpenShift runs containers as an arbitrary UID in group 0, so grant group 0 the same
# access as the owner (useradd's default home dir mode blocks non-owner traversal otherwise)
RUN chgrp -R 0 ${DEPLOY_USER_DIR} && chmod -R g=u ${DEPLOY_USER_DIR}

USER ${DEPLOY_USER}

# Do the deployment for GRIMOIRELAB_RELEASE

WORKDIR ${DEPLOY_USER_DIR}

RUN mkdir -p ${CONF_DIR} && \
    git clone -b ${GRIMOIRELAB_RELEASE} --depth 1 https://github.com/grimoirelab/grimoirelab.git && \
    cp grimoirelab/requirements.txt ${CONF_DIR}/requirements.txt && \
    echo ${GRIMOIRELAB_RELEASE} > ${DEPLOY_USER_DIR}/release && \
    echo "Installing and checking GrimoireLab Release"

# sudo requires PAM, which isn't configured in this base image; install as root instead
USER root
# scipy/sparse/sparsetools compiles heavy C++ templates that OOM at -O3 even single-threaded
RUN NPY_NUM_BUILD_JOBS=1 CFLAGS="-O1" CXXFLAGS="-O1" pip install -r ${CONF_DIR}/requirements.txt
USER ${DEPLOY_USER}

HEALTHCHECK --interval=60s --timeout=6s --retries=1 CMD healthcheck.py -c ${CONF_DIR}/setup.cfg -s 'Exception in Task Manager' || exit 1

CMD ["/bin/sh", "-c", "${DEPLOY_USER_DIR}/stage"]
