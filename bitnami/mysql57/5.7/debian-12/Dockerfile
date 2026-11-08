# syntax=docker/dockerfile:1.7
ARG IMAGE_REVISION=1
ARG MYSQL_VERSION=5.7.44
ARG MYSQL_MD5=1a637fce4599d9bf5f1c81699f086274
ARG OPENSSL_VERSION=1.1.1w
ARG OPENSSL_SHA256=cf3098950cb4d853ad95c0841f1f9c6d3dc102dccfcacd521d93925208b76ac8
ARG INI_FILE_VERSION=1.4.8
ARG GCC_VERSION=11
ARG BUILD_MAX_JOBS=4

FROM docker.io/library/golang:1.27-bookworm AS ini-file-builder
ARG INI_FILE_VERSION
WORKDIR /src
RUN set -eux; \
    curl -fsSL "https://github.com/bitnami/ini-file/archive/refs/tags/v${INI_FILE_VERSION}.tar.gz" \
      | tar -xz --strip-components=1; \
    CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /out/ini-file .; \
    mkdir -p /out/licenses; \
    cp LICENSE* /out/licenses/ini-file-${INI_FILE_VERSION}.txt

FROM docker.io/library/debian:bookworm AS openssl-builder
ARG OPENSSL_VERSION
ARG OPENSSL_SHA256
ARG BUILD_MAX_JOBS
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential perl ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
RUN set -eux; \
    F="openssl-${OPENSSL_VERSION}.tar.gz"; \
    TAG="OpenSSL_$(echo "${OPENSSL_VERSION}" | tr . _)"; \
    curl -fsSL -o "$F" "https://github.com/openssl/openssl/releases/download/${TAG}/$F"; \
    echo "${OPENSSL_SHA256}  $F" | sha256sum -c -; \
    tar -xzf "$F" --strip-components=1; \
    rm "$F"
RUN set -eux; \
    ./config --prefix=/opt/openssl --openssldir=/etc/ssl --libdir=lib shared no-tests; \
    JOBS="$(nproc)"; \
    if [ "$JOBS" -gt "${BUILD_MAX_JOBS}" ]; then JOBS="${BUILD_MAX_JOBS}"; fi; \
    make -j"${JOBS}"; \
    make install_sw; \
    rm -f /opt/openssl/lib/*.a; \
    cp LICENSE /opt/openssl/LICENSE

FROM docker.io/library/debian:bookworm AS mysql-builder
ARG MYSQL_VERSION
ARG MYSQL_MD5
ARG OPENSSL_VERSION
ARG GCC_VERSION
ARG BUILD_MAX_JOBS
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential gcc-${GCC_VERSION} g++-${GCC_VERSION} cmake bison pkg-config patchelf \
      ca-certificates curl libncurses-dev libaio-dev libsasl2-dev libldap2-dev libtirpc-dev \
    && rm -rf /var/lib/apt/lists/*
COPY --from=openssl-builder /opt/openssl /opt/openssl
WORKDIR /src
RUN set -eux; \
    F="mysql-boost-${MYSQL_VERSION}.tar.gz"; \
    curl -fsSL -o "$F" "https://cdn.mysql.com/Downloads/MySQL-5.7/$F" \
      || curl -fsSL -o "$F" "https://cdn.mysql.com/archives/mysql-5.7/$F"; \
    echo "${MYSQL_MD5}  $F" | md5sum -c -; \
    tar -xzf "$F" --strip-components=1; \
    rm "$F"
RUN set -eux; \
    export LD_LIBRARY_PATH=/opt/openssl/lib; \
    RPATH="-Wl,-rpath,/opt/bitnami/mysql/lib/private"; \
    mkdir build; \
    cd build; \
    cmake .. \
      -DCMAKE_C_COMPILER=gcc-${GCC_VERSION} \
      -DCMAKE_CXX_COMPILER=g++-${GCC_VERSION} \
      -DBUILD_CONFIG=mysql_release \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_INSTALL_PREFIX=/opt/bitnami/mysql \
      -DMYSQL_DATADIR=/bitnami/mysql/data \
      -DSYSCONFDIR=/opt/bitnami/mysql/conf \
      -DMYSQL_UNIX_ADDR=/opt/bitnami/mysql/tmp/mysql.sock \
      -DWITH_BOOST=/src/boost \
      -DDOWNLOAD_BOOST=0 \
      -DENABLE_DOWNLOADS=OFF \
      -DWITH_SSL=/opt/openssl \
      -DWITH_SASL=system \
      -DWITH_LDAP=system \
      -DWITH_PIC=ON \
      -DWITH_EMBEDDED_SERVER=OFF \
      -DWITH_UNIT_TESTS=OFF \
      -DINSTALL_MYSQLTESTDIR= \
      -DCMAKE_EXE_LINKER_FLAGS="${RPATH}" \
      -DCMAKE_SHARED_LINKER_FLAGS="${RPATH}" \
      -DCMAKE_MODULE_LINKER_FLAGS="${RPATH}"; \
    JOBS="$(nproc)"; \
    if [ "$JOBS" -gt "${BUILD_MAX_JOBS}" ]; then JOBS="${BUILD_MAX_JOBS}"; fi; \
    echo "building with ${JOBS} jobs"; \
    cmake --build . -j"${JOBS}"; \
    cmake --install . --strip; \
    mkdir -p /opt/bitnami/mysql/lib/private; \
    cp -L /opt/openssl/lib/libssl.so.1.1 /opt/openssl/lib/libcrypto.so.1.1 /opt/bitnami/mysql/lib/private/; \
    strip --strip-unneeded /opt/bitnami/mysql/lib/private/*.so.1.1; \
    patchelf --set-rpath '$ORIGIN' /opt/bitnami/mysql/lib/private/libssl.so.1.1 /opt/bitnami/mysql/lib/private/libcrypto.so.1.1; \
    rm -rf /opt/bitnami/mysql/man; \
    find /opt/bitnami/mysql/lib -maxdepth 1 -name '*.a' -delete; \
    mkdir -p /opt/bitnami/mysql/licenses; \
    cp /src/LICENSE /opt/bitnami/mysql/licenses/mysql-${MYSQL_VERSION}.txt; \
    cp /opt/openssl/LICENSE /opt/bitnami/mysql/licenses/openssl-${OPENSSL_VERSION}.txt

FROM docker.io/library/debian:bookworm-slim
ARG MYSQL_VERSION
ARG IMAGE_REVISION
ARG TARGETARCH

LABEL org.opencontainers.image.base.name="docker.io/library/debian:bookworm-slim" \
      org.opencontainers.image.description="MySQL ${MYSQL_VERSION} built from source with Bitnami container scripts" \
      org.opencontainers.image.licenses="Apache-2.0" \
      org.opencontainers.image.ref.name="${MYSQL_VERSION}-debian-12" \
      org.opencontainers.image.source="https://github.com/moechs/containers.git" \
      org.opencontainers.image.title="mysql" \
      org.opencontainers.image.version="${MYSQL_VERSION}"

ENV HOME="/" \
    OS_ARCH="${TARGETARCH:-amd64}" \
    OS_FLAVOUR="debian-12" \
    OS_NAME="linux"

COPY prebuildfs /
RUN find /usr/sbin -maxdepth 1 -type f \( -name install_packages -o -name run-script \) -exec chmod 0755 {} +
SHELL ["/bin/bash", "-o", "errexit", "-o", "nounset", "-o", "pipefail", "-c"]

RUN install_packages ca-certificates libaio1 libcrypt1 libgcc-s1 libsasl2-2 \
      libstdc++6 libtinfo6 libtirpc3 procps psmisc

COPY --from=mysql-builder /opt/bitnami/mysql /opt/bitnami/mysql
COPY --from=ini-file-builder /out/ini-file /opt/bitnami/common/bin/ini-file
COPY --from=ini-file-builder /out/licenses /opt/bitnami/common/licenses
COPY rootfs /

RUN set -eux; \
    find /opt/bitnami/scripts -type d -exec chmod 0755 {} +; \
    find /opt/bitnami/scripts -type f -name '*.sh' -exec chmod 0755 {} +; \
    mkdir -p /docker-entrypoint-initdb.d \
      /opt/bitnami/mysql/conf /opt/bitnami/mysql/conf.default \
      /opt/bitnami/mysql/logs /opt/bitnami/mysql/run \
      /opt/bitnami/mysql/tmp /opt/bitnami/mysql/var; \
    chmod -R g+rwX /opt/bitnami; \
    chmod 0755 /opt/bitnami/common/bin/ini-file; \
    apt-get update; \
    apt-get upgrade -y; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives; \
    find / -xdev -perm /6000 -type f -exec chmod a-s {} \; || true

RUN set -eu; \
    for f in /opt/bitnami/mysql/bin/* /opt/bitnami/mysql/lib/*.so* /opt/bitnami/mysql/lib/plugin/*.so /opt/bitnami/mysql/lib/private/*.so*; do \
      out="$(ldd "$f" 2>&1 || true)"; \
      case "$out" in *"not found"*) echo "missing libs: $f"; echo "$out"; exit 1;; esac; \
    done; \
    for b in mysqld mysql; do \
      out="$(ldd "/opt/bitnami/mysql/bin/$b")"; \
      case "$out" in *"/opt/bitnami/mysql/lib/private/libssl.so.1.1"*) ;; *) echo "$b is not using private OpenSSL"; echo "$out"; exit 1;; esac; \
    done; \
    test ! -e /opt/openssl; \
    /opt/bitnami/mysql/bin/mysqld --version; \
    t="$(mktemp)"; \
    printf '[mysqld]\n' > "$t"; \
    /opt/bitnami/common/bin/ini-file set --section mysqld --key port --value 3306 "$t"; \
    test "$(/opt/bitnami/common/bin/ini-file get --section mysqld --key port "$t")" = "3306"; \
    rm -f "$t"

RUN /opt/bitnami/scripts/mysql/postunpack.sh

ENV APP_VERSION="${MYSQL_VERSION}" \
    BITNAMI_APP_NAME="mysql" \
    IMAGE_REVISION="${IMAGE_REVISION}" \
    PATH="/opt/bitnami/common/bin:/opt/bitnami/mysql/bin:/opt/bitnami/mysql/sbin:$PATH"

EXPOSE 3306

USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/mysql/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/mysql/run.sh" ]
