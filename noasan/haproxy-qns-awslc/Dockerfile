FROM ubuntu:20.04 AS builder-ssl

ENV DEBIAN_FRONTEND noninteractive
RUN apt-get -y update && apt-get -y install git g++ make gcc wget autoconf libtool cmake curl

RUN git clone https://github.com/aws/aws-lc
RUN cd aws-lc && export GIMME_GO_VERSION=1.18 && curl -sL https://raw.githubusercontent.com/travis-ci/gimme/master/gimme > gimme && bash gimme && export GOROOT='/root/.gimme/versions/go1.18.linux.amd64' && export PATH="/root/.gimme/versions/go1.18.linux.amd64/bin:${PATH}" && cmake -DBUILD_SHARED_LIBS=1 -B build && make -C build && make -C build install

FROM ubuntu:20.04 AS builder

COPY --from=builder-ssl /usr/local/include/openssl/ /usr/local/include/openssl/
COPY --from=builder-ssl \
  /usr/local/lib/libssl.so* /usr/local/lib/libcrypto.so* /usr/local/lib/

ADD *.patch /tmp/
ADD https://api.github.com/repos/haproxytech/quic-dev/git/refs/heads/qns version.json
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get -y update && apt-get -y install git make gcc liblua5.3-0 liblua5.3-dev \
  && git clone https://github.com/haproxy/haproxy haproxy \
  && cd /haproxy \
  && patch -p1 < /tmp/0001-Add-timestamps-to-stderr-sink.patch \
  && make -j $(nproc) \
    CC=gcc \
    TARGET=linux-glibc \
    CPU=generic \
    USE_OPENSSL_AWSLC=1 \
    USE_QUIC=1 \
    SSL_INC=/usr/local/include/ \
    SSL_LIB=/usr/local/lib/ \
    SMALL_OPTS="" \
    DEBUG="-DDEBUG_DONT_SHARE_POOLS -DDEBUG_MEMORY_POOLS -DDEBUG_STRICT=2 -DDEBUG_TASK -DDEBUG_FAIL_ALLOC" \
    LDFLAGS="-fuse-ld=gold" \
    ARCH_FLAGS="-pg" \
    USE_LUA=1 LUA_LIB_NAME=lua5.3 \
    IGNOREGIT=1 VERSION=$(git log -1 --pretty=format:%H) \
  && make install

FROM martenseemann/quic-network-simulator-endpoint:latest

# Required for lighttpd
ENV TZ=Europe/Paris
RUN echo $TZ > /etc/timezone && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime
RUN apt-get -y update && apt-get -y install lighttpd liblua5.3-0 && rm -rf /var/lib/apt/lists/*

COPY --from=builder-ssl \
  /usr/local/lib/libssl.so* /usr/local/lib/libcrypto.so* /usr/local/lib/
COPY --from=builder /usr/local/sbin/haproxy /usr/local/sbin/
COPY quic.cfg lighttpd.cfg /
COPY sslkeylogger.lua /

COPY run_endpoint.sh .
RUN chmod +x run_endpoint.sh

STOPSIGNAL SIGUSR1

ENTRYPOINT [ "/run_endpoint.sh" ]
