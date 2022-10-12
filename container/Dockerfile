FROM quay.io/centos/centos:stream9

USER 0

ENV APP_ROOT=/opt/app-root \
    HOME=/opt/app-root/src \
    PLATFORM="el9"

RUN mkdir -p ${HOME}/.pki/nssdb && \
    chown -R 1001:0 ${HOME}/.pki &&  \
    yum install -y --setopt=tsflags=nodocs centos-release-nfs-ganesha4 && \
    yum install -y --setopt=tsflags=nodocs nfs-ganesha-vfs && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

# Copy extra files to the image.
COPY ./root/ /
COPY ganesha.conf /etc/ganesha/ganesha.conf

WORKDIR ${HOME}

# Reset permissions of modified directories and add default user
RUN rpm-file-permissions && \
    useradd -u 1001 -r -g 0 -d ${HOME} -s /sbin/nologin \
        -c "Default Application User" default && \
    chown -R 1001:0 ${APP_ROOT} && \
    mkdir -p /opt/app-root/ganesha && \
    mkdir -p /opt/app-root/ganesha/data && \
    mkdir -p /var/lib/nfs/ganesha && \
    fix-permissions ${APP_ROOT} -P && \
    fix-permissions /var/lib/nfs/ganesha -P

# Ganesha ports
EXPOSE 111 111/udp 2049 20048 38465-38467

USER 1001

CMD /usr/bin/ganesha.nfsd -F -L /proc/1/fd/1 -f /etc/ganesha/ganesha.conf -p /opt/app-root/ganesha/ganesha.pid