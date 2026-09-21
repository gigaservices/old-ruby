FROM debian:trixie-slim AS builder

ENV DEBIAN_FRONTEND=noninteractive \
    OPENSSL_VERSION=1.0.2u \
    OPENSSL_DOWNLOAD_SHA256=ecd0c6ffb493dd06707d38b14bb4d8c2288bb7033735606569d8f90f89669d16 \
    RUBY_MAJOR=2.3 \
    RUBY_VERSION=2.3.8 \
    RUBY_DOWNLOAD_SHA256=910f635d84fd0d81ac9bdee0731279e6026cb4cd1315bbbb5dfb22e09c5c1dfe

RUN apt-get update && apt-get install -y --no-install-recommends \
    autoconf \
    bison \
    ca-certificates \
    curl \
    dpkg-dev \
    gcc \
    libffi-dev \
    libgdbm-dev \
    libgmp-dev \
    libncurses-dev \
    libreadline-dev \
    libyaml-dev \
    libjemalloc-dev \
    make \
    tar \
    xz-utils \
    zlib1g-dev \
  && rm -rf /var/lib/apt/lists/*

RUN set -eux \ 
  && curl -fSL "https://www.openssl.org/source/old/1.0.2/openssl-${OPENSSL_VERSION}.tar.gz" -o openssl.tar.gz \
  && echo "${OPENSSL_DOWNLOAD_SHA256} openssl.tar.gz" | sha256sum -c - \
  && mkdir -p /usr/src/openssl \
  && tar -xzf openssl.tar.gz -C /usr/src/openssl --strip-components=1 \
  && rm openssl.tar.gz \
  && cd /usr/src/openssl \
  && ./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib \
  && make -j"$(nproc)" \
  && make install_sw \
  && rm -rf /usr/src/openssl

ENV RUBY_COMPAT_CFLAGS="-O3 -fno-strict-overflow -Wno-discarded-qualifiers -Wno-implicit-int -Wno-incompatible-pointer-types -Wno-int-conversion -Wno-return-type -std=gnu89 -Wno-error=implicit-function-declaration"

RUN set -eux \ 
  && curl -fSL "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-${RUBY_VERSION}.tar.xz" -o ruby.tar.xz \
  && echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c - \
  && mkdir -p /usr/src/ruby \
  && tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1 \
  && rm ruby.tar.xz \
  && cd /usr/src/ruby \
  && { \
    echo '#define ENABLE_PATH_CHECK 0'; \
    echo; \
    cat file.c; \
  } > file.c.new \
  && mv file.c.new file.c \
  && autoconf \
  && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
  && CFLAGS="$RUBY_COMPAT_CFLAGS" ./configure \
    --build="$gnuArch" \
    --prefix=/usr/local \
    --with-openssl-dir=/usr/local/ssl \
    --disable-install-doc \
    --enable-shared \
    --with-jemalloc \
  && make -j"$(nproc)" \
  && make install \
  && rm -rf /usr/src/ruby

FROM debian:trixie-slim

ENV RUBYGEMS_VERSION=3.0.3 \
    GEM_HOME=/usr/local/bundle

ENV BUNDLE_PATH="$GEM_HOME" \
    BUNDLE_SILENCE_ROOT_WARNING=1 \
    BUNDLE_APP_CONFIG="$GEM_HOME"
  
ENV PATH=$GEM_HOME/bin:$BUNDLE_PATH/gems/bin:/usr/local/ssl/bin:$PATH

RUN apt-get update && apt-get install -y --no-install-recommends \
    bzip2 \
    ca-certificates \
    libffi8 \
    libgdbm6 \
    libgmp10 \
    libreadline8 \
    libyaml-0-2 \
    libjemalloc2 \
    procps \
    zlib1g \
  && rm -rf /var/lib/apt/lists/* \
  && mkdir -p /usr/local/etc \
  && { \
    echo 'install: --no-document'; \
    echo 'update: --no-document'; \
  } >> /usr/local/etc/gemrc

COPY --from=builder /usr/local/ssl /usr/local/ssl
COPY --from=builder /usr/local /usr/local

ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt \
    SSL_CERT_DIR=/etc/ssl/certs

RUN set -ex \
  && echo "/usr/local/ssl/lib" > /etc/ld.so.conf.d/openssl-1.0.conf \
  && ldconfig \
  && rm -rf /usr/local/ssl/certs \
  && ln -s /etc/ssl/certs /usr/local/ssl/certs \
  && ln -sf /etc/ssl/certs/ca-certificates.crt /usr/local/ssl/cert.pem \
  && gem update --system "$RUBYGEMS_VERSION" \
  && rm -rf /root/.gem/ \
  && ruby --version && gem --version && bundle --version \
  && mkdir -p "$GEM_HOME" && chmod 777 "$GEM_HOME"

CMD [ "irb" ]