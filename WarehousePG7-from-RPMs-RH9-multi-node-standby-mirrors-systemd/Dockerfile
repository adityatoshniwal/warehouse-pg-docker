# syntax=docker/dockerfile:1

# define arguments as global variables
# ENV settings are scoped locally per build stage, by design
ARG GLOB_MAINTAINER="andreas.scherbaum@enterprisedb.com"
ARG GLOB_DESCRIPTION="Dockerfile to run WarehousePG 7.x"
ARG GLOB_IMAGE_TITLE="WarehousePG"
ARG GLOB_IMAGE_DESCRIPTION="WarehousePG Database image"
ARG GLOB_IMAGE_VERSION="0.1.1"
ARG GLOB_IMAGE_AUTHORS="andreas.scherbaum@enterprisedb.com"
ARG GLOB_COORDINATOR_DATA_DIRECTORY=/whpgdata/coordinator/whpgmne-1
ARG GLOB_WHPGHOME="/usr/local/greenplum-db"


#######################################################################
# use Rocky Linux as base image
FROM rockylinux:9 AS build-stage
ENV container=docker

ARG GLOB_MAINTAINER
ARG GLOB_DESCRIPTION
ARG GLOB_IMAGE_TITLE
ARG GLOB_IMAGE_DESCRIPTION
ARG GLOB_IMAGE_VERSION
ARG GLOB_IMAGE_AUTHORS
ARG GLOB_OS_NAME
ARG GLOB_OS_VERSION
ARG GLOB_BUILD_VERSION
ARG GLOB_WHPGHOME
#ARG LD_LIBRARY_PATH
ARG GLOB_COORDINATOR_DATA_DIRECTORY

# from commandline arguments
ARG EDBTOKEN
ARG EDBREPOSITORY

# metadata
LABEL maintainer=$GLOB_MAINTAINER
LABEL description=$GLOB_DESCRIPTION
LABEL org.opencontainers.image.title=$GLOB_IMAGE_TITLE
LABEL org.opencontainers.image.description=$GLOB_IMAGE_DESCRIPTION
LABEL org.opencontainers.image.version=$GLOB_BUILD_VERSION
LABEL org.opencontainers.image.authors=$GLOB_IMAGE_AUTHORS


# set environment variables
# start with C locale here, change to US later
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8
# configure environment for WarehousePG
ENV OS_NAME=$GLOB_OS_NAME
ENV OS_VERSION=$GLOB_OS_VERSION
ENV BUILD_VERSION=$GLOB_BUILD_VERSION
ENV WHPGHOME=$GLOB_WHPGHOME
#ENV LD_LIBRARY_PATH=$WHPGHOME/lib:$LD_LIBRARY_PATH
ENV COORDINATOR_DATA_DIRECTORY=$GLOB_COORDINATOR_DATA_DIRECTORY
ENV PATH=$WHPGHOME/bin:$PATH


# download setup.rpm.sh early on, which will fail early if the token doesn't work
RUN --mount=type=secret,id=edbtoken_secret --mount=type=secret,id=edbrepository_secret curl -u token:$(cat /run/secrets/edbtoken_secret) -1sLf https://downloads.enterprisedb.com/basic/$(cat /run/secrets/edbrepository_secret)/setup.rpm.sh -o /root/setup.rpm.sh


# install required packages for building WarehousePG
#RUN localectl set-locale LANG=en_US.UTF-8
RUN dnf update -y
RUN dnf install -y epel-release
RUN dnf update -y
#RUN dnf install -y sudo
#RUN dnf install -y passwd

COPY rpm-packages-build-step.txt /root/rpm-packages-build-step.txt
RUN dnf install -y $(cat /root/rpm-packages-build-step.txt)

RUN dnf clean all

ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8


# add gpadmin user for building the database
RUN groupadd gpadmin
RUN useradd -m -d /home/gpadmin -s /bin/bash -c "WarehousePG Database" -g gpadmin gpadmin
RUN echo "gpadmin ALL=(ALL) NOPASSWD: ALL" | sudo EDITOR='tee -a' visudo


# install the database from RPM
RUN chmod u+x /root/setup.rpm.sh
RUN /root/setup.rpm.sh

# installation goes into /usr/local/greenplum-db-<version>,
# and a symlink to /usr/local/greenplum-db is created
RUN dnf install -y warehouse-pg-7
# clients are already included in the server package
#RUN dnf install -y warehouse-pg-clients
# installation goes into /usr/local/bin
RUN dnf install -y whpg-backup


# switch to gpadmin user, do not build as root
USER gpadmin


#######################################################################
# database installed, but not started
# build a new image, without the RPMs
FROM rockylinux:9 AS image-stage
ENV container=docker

ARG GLOB_MAINTAINER
ARG GLOB_DESCRIPTION
ARG GLOB_IMAGE_TITLE
ARG GLOB_IMAGE_DESCRIPTION
ARG GLOB_IMAGE_VERSION
ARG GLOB_IMAGE_AUTHORS
ARG GLOB_OS_NAME
ARG GLOB_OS_VERSION
ARG GLOB_BUILD_VERSION
ARG GLOB_WHPGHOME
ARG GLOB_COORDINATOR_DATA_DIRECTORY

# metadata
LABEL maintainer=$GLOB_MAINTAINER
LABEL description=$GLOB_DESCRIPTION
LABEL org.opencontainers.image.title=$GLOB_IMAGE_TITLE
LABEL org.opencontainers.image.description=$GLOB_IMAGE_DESCRIPTION
LABEL org.opencontainers.image.version=$GLOB_BUILD_VERSION
LABEL org.opencontainers.image.authors=$GLOB_IMAGE_AUTHORS

# set environment variables
# start with C locale here, change to US later
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8
# configure environment for WarehousePG
ENV OS_NAME=$GLOB_OS_NAME
ENV OS_VERSION=$GLOB_OS_VERSION
ENV BUILD_VERSION=$GLOB_BUILD_VERSION
ENV WHPGHOME=$GLOB_WHPGHOME
ENV COORDINATOR_DATA_DIRECTORY=$GLOB_COORDINATOR_DATA_DIRECTORY
ENV PATH=$WHPGHOME/bin:$PATH
# LD_LIBRARY_PATH is set further down, once the RPMs are copied in from
# build-stage — see the comment there for why it can't be set here

# install required packages for running WarehousePG
#RUN localectl set-locale LANG=en_US.UTF-8
RUN dnf update -y
RUN dnf groupinstall -y "Development Tools"
RUN dnf install -y epel-release
RUN dnf update -y

COPY rpm-packages-image-step.txt /root/rpm-packages-image-step.txt
RUN dnf install -y $(cat /root/rpm-packages-image-step.txt)

RUN dnf install -y --enablerepo=crb libyaml libyaml-devel libuv-devel perl-IPC-Run

RUN dnf clean all

ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8

# run systemd as PID 1: strip unit dirs that don't apply/break in a container,
# keep systemd-tmpfiles-setup.service (needed to populate /run etc. at boot)
RUN (cd /lib/systemd/system/sysinit.target.wants/ && \
     for i in *; do [ "$i" = "systemd-tmpfiles-setup.service" ] || rm -f "$i"; done) && \
    rm -f /lib/systemd/system/multi-user.target.wants/* \
          /etc/systemd/system/*.wants/* \
          /lib/systemd/system/local-fs.target.wants/* \
          /lib/systemd/system/sockets.target.wants/*udev* \
          /lib/systemd/system/sockets.target.wants/*initctl* \
          /lib/systemd/system/basic.target.wants/* \
          /lib/systemd/system/anaconda.target.wants/*

# re-enable dbus socket (wiped by the multi-user.target.wants/* cleanup above)
RUN ln -sf /usr/lib/systemd/system/dbus.socket \
           /usr/lib/systemd/system/sockets.target.wants/dbus.socket

# sshd needs host keys generated at build time
RUN ssh-keygen -A


# add WarehousePG user for building and running the database
RUN groupadd gpadmin
RUN useradd -m -d /home/gpadmin -s /bin/bash -c "WarehousePG Database" -g gpadmin gpadmin
RUN echo "gpadmin ALL=(ALL) NOPASSWD: ALL" | sudo EDITOR='tee -a' visudo
RUN echo "gpadmin:whpg5432" | chpasswd
RUN echo "root:whpg5432" | chpasswd


# expose ports
EXPOSE 22/tcp
EXPOSE 5432/tcp
EXPOSE 6000-6010/tcp
EXPOSE 7000-7010/tcp
EXPOSE 8000-8010/tcp
EXPOSE 9000-9010/tcp
EXPOSE 30000-30010/tcp


# copy installation over from the build stage image
RUN rm -rf /usr/local/* /usr/edb/*
COPY --from=build-stage /usr/local /usr/local
COPY --from=build-stage /usr/edb /usr/edb

# set only now (after the RPMs are copied in), so it can't shadow system libs
# for dnf/systemctl/etc. earlier in this stage. Baked into the container env,
# so tools like psql resolve libpq.so.5 immediately for any `docker exec`
# shell, without racing whpg-cluster-init.service's ~/.bashrc.d population.
ENV LD_LIBRARY_PATH=${WHPGHOME}/lib

# apparently this does not work everywhere
# and breaks on AWS
#COPY --chown=root:root --chmod=0644 sysctl.conf /etc/sysctl.d/50-warehousepg.conf
#COPY --chown=root:root --chmod=0644 limits.conf /etc/security/limits.d/50-warehousepg.conf


# switch to gpadmin user, do not build image as root
USER gpadmin

# we need a better way to make these modules available in the gpadmin Python installation
# gpstart/gpstop etc. run under the python3.11 that greenplum_path.sh puts first on
# PATH (ext/python3.11/bin), not the system python3 (3.9) that plain pip3 targets -
# so install into python3.11 explicitly (it ships without pip, hence ensurepip first)
RUN sudo bash -c ". ${WHPGHOME}/greenplum_path.sh && python3.11 -m ensurepip --upgrade"
RUN sudo bash -c ". ${WHPGHOME}/greenplum_path.sh && python3.11 -m pip install psycopg2"
RUN sudo bash -c ". ${WHPGHOME}/greenplum_path.sh && python3.11 -m pip install psutil"

# Create data directories
RUN sudo mkdir -p /whpgdata/coordinator /whpgdata/segments /whpgdata/segments/whpgdata1 /whpgdata/segments/whpgdata2 && \
    sudo chown -R gpadmin:gpadmin /whpgdata


COPY --chown=gpadmin:gpadmin hostfile_whpginitsystem7 /home/gpadmin/hostfile_whpginitsystem
COPY --chown=gpadmin:gpadmin hostfile_segments_whpginitsystem7 /home/gpadmin/hostfile_segments_whpginitsystem
COPY --chown=gpadmin:gpadmin hostfile_ssh_whpginitsystem7 /home/gpadmin/hostfile_ssh_whpginitsystem
COPY --chown=gpadmin:gpadmin whpginitsystem_multinode7 /home/gpadmin/whpginitsystem_multinode
# the chmod option needs buildx
COPY --chown=gpadmin:gpadmin --chmod=0750 warehousepg_init7.sh /home/gpadmin/warehousepg_init.sh
COPY --chown=gpadmin:gpadmin --chmod=0755 list-seg-ports.sh /home/gpadmin/list-seg-ports.sh
COPY --chown=gpadmin:gpadmin --chmod=0750 cmd.sh /home/gpadmin/cmd.sh

# bake in sshd config that cmd.sh used to set at runtime, and register the
# cluster-bootstrap logic as a systemd unit (sshd.service now starts itself)
COPY --chown=root:root --chmod=0644 sshd-local.conf /etc/ssh/sshd_config.d/60-local.conf
COPY --chown=root:root --chmod=0644 whpg-cluster-init.service /etc/systemd/system/whpg-cluster-init.service
RUN sudo systemctl enable sshd.service whpg-cluster-init.service

WORKDIR /home/gpadmin

# PID 1 must be root; cmd.sh itself still runs as gpadmin via the systemd unit's User=
USER root

# ensure `docker stop`/`compose stop` trigger systemd's clean shutdown, not a hard kill
STOPSIGNAL SIGRTMIN+3

CMD ["/usr/sbin/init"]
