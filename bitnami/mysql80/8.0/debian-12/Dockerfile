# syntax=docker/dockerfile:1.7
ARG IMAGE_REVISION=1
ARG MYSQL_VERSION=8.0.46
ARG MYSQL_MD5=74cf76377b30f68815b3a711cd6bf715
ARG INI_FILE_VERSION=1.4.8
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

FROM docker.io/library/debian:bookworm AS mysql-builder
ARG MYSQL_VERSION
ARG MYSQL_MD5
ARG BUILD_MAX_JOBS
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential cmake bison pkg-config ca-certificates curl \
      libssl-dev libncurses-dev libaio-dev libtirpc-dev rpcsvc-proto \
      libudev-dev libsasl2-dev libldap2-dev libkrb5-dev \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
RUN set -eux; \
    F="mysql-boost-${MYSQL_VERSION}.tar.gz"; \
    curl -fsSL -o "$F" "https://cdn.mysql.com/Downloads/MySQL-8.0/$F" \
      || curl -fsSL -o "$F" "https://cdn.mysql.com/archives/mysql-8.0/$F"; \
    echo "${MYSQL_MD5}  $F" | md5sum -c -; \
    tar -xzf "$F" --strip-components=1; \
    rm "$F"
RUN set -eux; \
    mkdir build; \
    cd build; \
    cmake .. \
      -DBUILD_CONFIG=mysql_release \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_INSTALL_PREFIX=/opt/bitnami/mysql \
      -DMYSQL_DATADIR=/bitnami/mysql/data \
      -DSYSCONFDIR=/opt/bitnami/mysql/conf \
      -DMYSQL_UNIX_ADDR=/opt/bitnami/mysql/tmp/mysql.sock \
      -DWITH_BOOST=/src/boost \
      -DWITH_SSL=system \
      -DWITH_SASL=system \
      -DWITH_LDAP=system \
      -DWITH_KERBEROS=system \
      -DWITH_AUTHENTICATION_CLIENT_PLUGINS=ON \
      -DWITH_PACKAGE_FLAGS=ON \
      -DWITH_UNIT_TESTS=OFF \
      -DINSTALL_MYSQLTESTDIR=; \
    JOBS="$(nproc)"; \
    if [ "$JOBS" -gt "${BUILD_MAX_JOBS}" ]; then JOBS="${BUILD_MAX_JOBS}"; fi; \
    echo "building with ${JOBS} jobs"; \
    cmake --build . -j"${JOBS}"; \
    cmake --install . --strip; \
    rm -rf /opt/bitnami/mysql/man /opt/bitnami/mysql/include; \
    find /opt/bitnami/mysql/lib -maxdepth 1 -name '*.a' -delete; \
    mkdir -p /opt/bitnami/mysql/licenses; \
    cp /src/LICENSE /opt/bitnami/mysql/licenses/mysql-${MYSQL_VERSION}.txt

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

RUN install_packages ca-certificates libaio1 libcom-err2 libgcc-s1 libgssapi-krb5-2 \
      libk5crypto3 libkeyutils1 libkrb5-3 libkrb5support0 libsasl2-2 libssl3 \
      libstdc++6 libtinfo6 libtirpc3 libudev1 procps psmisc

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
    for f in /opt/bitnami/mysql/bin/* /opt/bitnami/mysql/lib/plugin/*.so /opt/bitnami/mysql/lib/private/*.so*; do \
      out="$(ldd "$f" 2>&1 || true)"; \
      case "$out" in *"not found"*) echo "missing libs: $f"; echo "$out"; exit 1;; esac; \
    done; \
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
