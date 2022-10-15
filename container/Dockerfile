FROM quay.io/centos/centos:stream9 as build

# Build ganesha from source to minimize size with only VFS, install it to /usr/local
# and a use multi stage build to have a smaller image
# Set NFS_V4_RECOV_ROOT to /export

USER 0

RUN INSTALL_PKGS =  "bison \
                    cmake \
                    dbus-devel \
                    flex \
                    tar \
                    gcc \
                    gcc-c++ \
                    git \
                    jemalloc-devel \
                    krb5-devel \
                    libblkid-devel \
                    libnfsidmap-devel \
                    libnsl2-devel \
                    libntirpc-devel \
                    libuuid-devel \
                    patch \
                    userspace-rcu-devel \
                    xfsprogs-devel"  && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

# Clone specific version of ganesha
# build ganesha only supporting nfsv4 and vfs
# Set NFS_V4_RECOV_ROOT to /tmp we don't support recovery in this release
# we disable dbus (-DUSE_DBUS=OFF) for the single share manager since we don't use dynamic exports
ARG GANESHA_VERSION=V4.0.9
RUN git clone --branch ${GANESHA_VERSION} --recurse-submodules https://github.com/nfs-ganesha/nfs-ganesha
WORKDIR /nfs-ganesha
RUN mkdir -p /usr/local \
    && cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_CONFIG=vfs_only \
    -DUSE_DBUS=OFF -DUSE_NFS3=OFF -DUSE_NLM=OFF -DUSE_RQUOTA=OFF -DUSE_9P=OFF -D_MSPAC_SUPPORT=OFF -DRPCBIND=OFF \
    -DUSE_RADOS_RECOV=OFF -DRADOS_URLS=OFF -DUSE_FSAL_VFS=ON -DUSE_FSAL_XFS=OFF \
    -DUSE_FSAL_PROXY_V4=OFF -DUSE_FSAL_PROXY_V3=OFF -DUSE_FSAL_LUSTRE=OFF -DUSE_FSAL_LIZARDFS=OFF \
    -DUSE_FSAL_KVSFS=OFF -DUSE_FSAL_CEPH=OFF -DUSE_FSAL_GPFS=OFF -DUSE_FSAL_PANFS=OFF -DUSE_FSAL_GLUSTER=OFF \
    -DUSE_GSS=NO \
    -DCMAKE_INSTALL_PREFIX=/usr/local src/ \
	  && make \
	  && make install
RUN mkdir -p /ganesha-extra \
    && mkdir -p /ganesha-extra/etc/dbus-1/system.d \
    && cp src/scripts/ganeshactl/org.ganesha.nfsd.conf /ganesha-extra/etc/dbus-1/system.d/

FROM quay.io/centos/centos:stream9 as run

RUN INSTALL_PKGS =  "hostname \
                    jemalloc \
                    libblkid \
                    libnfsidmap \
                    libntirpc \
                    libuuid \
                    nfs-utils \
                    rpcbind \
                    userspace-rcu \
                    xfsprogs \"  && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

RUN mkdir -p /var/run/dbus \
    && mkdir -p /export

# add libs from /usr/local/lib64
RUN echo /usr/local/lib64 > /etc/ld.so.conf.d/local_libs.conf

# do not ask systemd for user IDs or groups (slows down dbus-daemon start)
RUN sed -i s/systemd// /etc/nsswitch.conf

# ganesha reads /etc/mtab for mounted volumes
RUN ln -sf /proc/self/mounts /etc/mtab

COPY --from=build /usr/local /usr/local/
COPY --from=build /ganesha-extra /

ARG binary=nfs-provisioner
COPY ${binary} /nfs-provisioner

# run ldconfig after libs have been copied
RUN ldconfig

# expose mountd 20048/tcp and nfsd 2049/tcp and rpcbind 111/tcp 111/udp
EXPOSE 2049/tcp 20048/tcp 111/tcp 111/udp

ENTRYPOINT ["/nfs-provisioner"]










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