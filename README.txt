◆docker-compose-dev
version: '3'
services:
  app:
    build: .
    dns:
      - 8.8.8.8
      - 8.8.4.4
    ports:
      - "25022:22"
    volumes:
      - ".:/go/src/app"
      - "appuser_home:/home/appuser"
volumes:
  appuser_home:
    driver: local


◆entrypoint
#!/usr/bin/env bash
set -e

chown -R appuser:appuser /home/appuser
chown -R appuser:appuser /go/src/app

exec "$@"


◆Dockerfile
#
# base
#
FROM golang:1.12.7-buster as base

ENV GO111MODULE=on

# add app-user
ARG app_uid=10001

RUN adduser -u ${app_uid} --disabled-password --no-create-home --gecos "" appuser && \
  mkdir -p /go/src/app && \
  chmod 755 /go/src/app && \
  chown -R appuser:appuser /go/src/app

#
# dev
#
FROM base as dev

# development tools
RUN apt-get update && apt-get install -y --no-install-recommends \
  vim-tiny \
  unzip \
  jq \
  openssh-server \
  dnsutils \
  traceroute \
  strace \
  tcpdump \
  bash-completion \
  sudo \
  gosu && \
  rm -rf /var/lib/apt/lists/*

# ssh settings for IDE
RUN mkdir -p /var/run/sshd && \
  #sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
  #echo 'X11UseLocalhost no' >>/etc/ssh/sshd_config && \
  #echo 'Ciphers  3des-cbc,aes128-cbc,aes192-cbc,aes256-cbc,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com,arcfour,arcfour128,arcfour256,blowfish-cbc,cast128-cbc,chacha20-poly1305@openssh.com' >>/etc/ssh/sshd_config && \
  sed 's~session\s*required\s*pam_loginuid.so~session optional pam_loginuid.so~g' -i /etc/pam.d/sshd

# add sudoers
RUN echo "appuser ALL=(ALL) NOPASSWD: ALL" >/etc/sudoers.d/appuser

# volume mount and chown script
VOLUME ["/home/appuser", "/go/src/app"]

COPY dev-entrypoint.sh /usr/local/bin
RUN chmod +x /usr/local/bin/dev-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/dev-entrypoint.sh"]

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]

#
# build
#
#FROM base as builder

#WORKDIR /go/src/app
#COPY go.mod go.sum ./

#
# release
#
#FROM base
#COPY --from=builder
#



[EOF]
