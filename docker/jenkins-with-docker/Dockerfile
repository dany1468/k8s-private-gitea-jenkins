FROM jenkins/jenkins:lts

ARG user=jenkins
ENV DEBIAN_FRONTEND noninteractive

ARG DOCKER_CLI_VERSION=19.03.5
ARG DOCKER_HOST_GID=999

USER root

# install prerequirement tools, and upgrade
RUN apt-get update -y \
 && apt-get upgrade -y \
 && apt-get install -y --no-install-recommends gnupg apt-utils apt-transport-https \
 && rm -rf /var/lib/apt/lists/*

 # install docker client
RUN curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-${DOCKER_CLI_VERSION}.tgz | tar -xz -C /tmp \
  && mv /tmp/docker/docker /usr/local/bin \
  && rm -r /tmp/docker*
RUN groupadd -g ${DOCKER_HOST_GID} docker
RUN usermod -aG docker jenkins

USER ${user}
