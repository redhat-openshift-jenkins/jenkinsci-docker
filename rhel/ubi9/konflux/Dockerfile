FROM quay.io/redhat-user-workloads/rh-openshift-builds-tenant/openshift-jenkins-2-462/go-init-2-462@sha256:8244f9bfe9da17b34c83d5bad6f031f623c49e353abb0dd38249eb3bc53cb622 as go-init

FROM quay.io/redhat-user-workloads/rh-openshift-builds-tenant/openshift-jenkins-2-462/jenkins-war@sha256:de13a4065d020d8d5e68d5642d6a2f606093600d9e91a6a65dedf88b2a0c47df as jenkins-war

FROM quay.io/redhat-user-workloads/rh-openshift-builds-tenant/openshift-jenkins-2-462/jenkins-plugin-manager-2-462@sha256:538cf5c12819550d2565a5132daef6fcf33a25b4328e37d345504a769fc3e30c as plugin-manager

FROM registry.access.redhat.com/ubi9/openjdk-17-runtime:1.20-2.1725851029 AS controller

USER root

ENV LANG=C.UTF-8

ARG TARGETARCH
ARG COMMIT_SHA

# ubi9/openjdk-*-runtime images are based on ubi-minimal, which uses microdnf.
RUN microdnf install --disableplugin=subscription-manager --setopt=install_weak_deps=0 --setopt=tsflags=nodocs -y \
        fontconfig \
        freetype \
        git \
        git-lfs \
        unzip \
        which \
        tzdata \
    && microdnf clean --disableplugin=subscription-manager all

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG http_port=8080
ARG agent_port=50000
ARG JENKINS_HOME=/var/jenkins_home
ARG REF=/usr/share/jenkins/ref

ENV JENKINS_HOME=$JENKINS_HOME
ENV JENKINS_SLAVE_AGENT_PORT=${agent_port}
ENV REF=$REF

# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN mkdir -p $JENKINS_HOME \
  && chown ${uid}:${gid} $JENKINS_HOME \
  && groupadd -g ${gid} ${group} \
  && useradd -N -d "$JENKINS_HOME" -u ${uid} -g ${gid} -l -m -s /bin/bash ${user}

# Jenkins home directory is a volume, so configuration and build history
# can be persisted and survive image upgrades
VOLUME $JENKINS_HOME

# $REF (defaults to `/usr/share/jenkins/ref/`) contains all reference configuration we want
# to set on a fresh new installation. Use it to bundle additional plugins
# or config file with your custom jenkins Docker image.
RUN mkdir -p ${REF}/init.groovy.d

# Replace upstream use of tini with go-init.
# chown to root:root so the "jenkins" user cannot modify.
COPY --from=go-init --chown=root:root /usr/bin/go-init /usr/bin/go-init

# Extract jenkins.war from the container image build of core Jenkins
# chown to root:root so the "jenkins" user cannot modify
COPY --from=jenkins-war --chown=root:root /deployments/jenkins.war /usr/share/jenkins/jenkins.war

ENV JENKINS_UC=https://updates.jenkins.io
ENV JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental
ENV JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals
RUN chown -R ${user} "$JENKINS_HOME" "$REF"

# Extract plugin manager from the component image build
# chown to root:root so the "jenkins" user cannot modify
COPY --from=plugin-manager --chown=root:root /deployments/jenkins-plugin-manager.jar /opt/jenkins-plugin-manager.jar

# for main web interface:
EXPOSE ${http_port}

# will be used by attached agents:
EXPOSE ${agent_port}

ENV COPY_REFERENCE_FILE_LOG=$JENKINS_HOME/copy_reference_file.log

USER ${user}
# UBI9 JDK images use /home/default as a working directory, which is owned by the "default" user.
# Upstream Jenkins images use "/" as the working directory by default.
WORKDIR "/"
# UBI9 JDK images set the HOME env var to /home/default, which is owned by the "default" user.
# Update HOME env var to match JENKINS_HOME.
ENV HOME=$JENKINS_HOME

COPY jenkins-support /usr/local/bin/jenkins-support
COPY jenkins.sh /usr/local/bin/jenkins.sh
COPY jenkins-plugin-cli.sh /bin/jenkins-plugin-cli

# Use go-init instead of tini for ENTRYPOINT
ENTRYPOINT ["/usr/bin/go-init", "-main", "/usr/local/bin/jenkins.sh"]

# metadata labels
LABEL \
    org.opencontainers.image.vendor="Red Hat" \
    org.opencontainers.image.title="Red Hat Build of Jenkins Container Image" \
    org.opencontainers.image.description="The Jenkins Continuous Integration and Delivery server. Built by Red Hat." \
    org.opencontainers.image.version="2.462" \
    org.opencontainers.image.url="https://www.jenkins.io/" \
    org.opencontainers.image.source="https://github.com/redhat-openshift-jenkins/jenkinsci-docker" \
    org.opencontainers.image.licenses="MIT"
