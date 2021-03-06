FROM golang:1.16.0-alpine3.13 AS go_builder_

# Install dependencies
RUN apk add --no-cache build-base git make

# Create directory to store built artifacts
ARG INSTALL_DIR="/install"
RUN mkdir -p ${INSTALL_DIR}

ARG BUILDKIT_CLI_FOR_KUBECTL_VERSION="v0.1.2"
RUN git clone -b ${BUILDKIT_CLI_FOR_KUBECTL_VERSION} --single-branch --depth 1 https://github.com/vmware-tanzu/buildkit-cli-for-kubectl.git ${GOPATH}/src/github.com/vmware-tanzu/buildkit-cli-for-kubectl \
    && make -C ${GOPATH}/src/github.com/vmware-tanzu/buildkit-cli-for-kubectl ${INSTALL_DIR}/linux/kubectl-build BIN_DIR=${INSTALL_DIR} VERSION=${BUILDKIT_CLI_FOR_KUBECTL_VERSION}


FROM alpine:3.13.2 AS base_

# Make directory to store artifacts
RUN mkdir -p /flyteorg/bin /flyteorg/share

# Install k3s
ARG K3S_VERSION="v1.20.2%2Bk3s1"
RUN wget -q -O /flyteorg/bin/k3s https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION}/k3s \
    && chmod +x /flyteorg/bin/k3s

# Install buildkit-cli-for-kubectl
COPY --from=go_builder_ /install/linux/ /flyteorg/bin/

# Download flyte manifest
COPY deployment/sandbox/flyte_generated.yaml /flyteorg/share/flyte_generated.yaml

# Copy scripts
COPY docker/sandbox/kubectl docker/sandbox/wait-for-flyte.sh /flyteorg/bin/


FROM base_ AS default

# Install dependencies
RUN apk add --no-cache bash git make tini

# Copy entrypoints
COPY docker/sandbox/flyte-entrypoint-default.sh /flyteorg/bin/flyte-entrypoint.sh

# Update PATH variable
ENV PATH "/flyteorg/bin:${PATH}"

# Declare volumes for k3s
VOLUME /var/lib/kubelet
VOLUME /var/lib/rancher/k3s
VOLUME /var/lib/cni
VOLUME /var/log

# Expose Flyte ports
EXPOSE 30081 30084

ENTRYPOINT ["tini", "flyte-entrypoint.sh"]


FROM docker:20.10.3-dind AS dind

# Install dependencies
RUN apk add --no-cache bash git make tini

# Copy artifacts from base
COPY --from=base_ /flyteorg/ /flyteorg/

# Copy entrypoints
COPY docker/sandbox/flyte-entrypoint-dind.sh /flyteorg/bin/flyte-entrypoint.sh

# Update PATH variable
ENV PATH "/flyteorg/bin:${PATH}"

# Declare volumes for k3s
VOLUME /var/lib/kubelet
VOLUME /var/lib/rancher/k3s
VOLUME /var/lib/cni
VOLUME /var/log

# Expose Flyte ports
EXPOSE 30081 30084

ENTRYPOINT ["tini", "flyte-entrypoint.sh"]
