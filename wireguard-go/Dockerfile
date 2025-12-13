##
## WolfGuard-go (wireguard-go + wolfCrypt FIPS) Dockerfile
##
## This Dockerfile follows the steps in `wireguard-go/README.md`:
## - Build/install wolfSSL from a wolfSSL FIPS bundle using:
##     ./configure --enable-fips=v5 --enable-sp="yes, 256"
##     make; ./wolfcrypt/test/testwolfcrypt; ./fips-hash.sh; make; ./wolfcrypt/test/testwolfcrypt; make install
## - Clone wireguard-go at commit 12269c2761734b1 and apply `Wireguard-Go-FIPS-wolfCrypt-port.patch`
## - Vendor `go-wolfssl` locally and run tests, then `make` to produce `wolfguard-go`
##
## IMPORTANT (FIPS bundle):
## - The wolfSSL FIPS bundle is not generally publicly cloneable.
## - Place your wolfSSL FIPS bundle sources at: `wireguard-go/deps/wolfssl-fips/`
##   so Docker can `COPY` them into the builder image.
##
## Build from repo root (recommended):
##   docker build -f wireguard-go/Dockerfile -t wolfguard-go:fips .
##
## Run (needs /dev/net/tun + NET_ADMIN):
##   docker run --rm -it --cap-add=NET_ADMIN --device /dev/net/tun wolfguard-go:fips wg0
##

FROM golang:1.22-bookworm AS builder

ARG DEBIAN_FRONTEND=noninteractive

# Use bash for better error handling/logging in RUN steps
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Allow using forks without editing this Dockerfile
ARG WOLFSSL_REPO=https://github.com/wolfSSL/wolfssl.git
ARG GO_WOLFSSL_REPO=https://github.com/wolfSSL/go-wolfssl.git
ARG WIREGUARD_GO_REPO=https://github.com/WireGuard/wireguard-go.git

# WireGuard-go commit this patch targets (from README)
ARG WIREGUARD_GO_COMMIT=12269c2761734b1

WORKDIR /src

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    git \
    patch \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# --- wolfSSL build (matches your known-good command sequence) ---
RUN set -eux \
    && git clone --depth=1 "${WOLFSSL_REPO}" /src/wolfssl \
    && cd /src/wolfssl \
    && ./autogen.sh \
    && ./configure --enable-sp="yes, 256" \
    && make -j"$(nproc)" \
    && make install \
    && ldconfig

# --- wireguard-go + wolfCrypt FIPS port ---
WORKDIR /src
COPY wireguard-go/Wireguard-Go-FIPS-wolfCrypt-port.patch /src/patches/Wireguard-Go-FIPS-wolfCrypt-port.patch

RUN git clone "${WIREGUARD_GO_REPO}" /src/wireguard-go \
    && cd /src/wireguard-go \
    && git checkout "${WIREGUARD_GO_COMMIT}" \
    && cp /src/patches/Wireguard-Go-FIPS-wolfCrypt-port.patch ./Wireguard-Go-FIPS-wolfCrypt-port.patch \
    && patch -p1 < ./Wireguard-Go-FIPS-wolfCrypt-port.patch

# --- go-wolfssl (local replacement) ---
# In the README this is done via:
#   go get -u github.com/wolfssl/go-wolfssl
#   go mod edit -replace github.com/wolfssl/go-wolfssl=<path>
#
# For a repeatable Docker build we clone go-wolfssl locally and use a replace.
RUN git clone "${GO_WOLFSSL_REPO}" /src/go-wolfssl

# Keep go-wolfssl options in sync with the wolfSSL feature set we built.
# See: https://github.com/wolfSSL/go-wolfssl
RUN set -eux \
    && cd /src/go-wolfssl \
    && if [ -x ./generateOptions.sh ]; then \
         ./generateOptions.sh /src/wolfssl; \
       elif [ -x ./generateOptions ]; then \
         ./generateOptions /src/wolfssl; \
       else \
         echo "Missing generateOptions(.sh) in go-wolfssl checkout" >&2; \
         exit 1; \
       fi

# Ensure cgo can find wolfSSL headers/libs from /usr/local
ENV CGO_ENABLED=1 \
    CGO_CFLAGS="-I/usr/local/include" \
    CGO_LDFLAGS="-L/usr/local/lib" \
    LD_LIBRARY_PATH="/usr/local/lib"

WORKDIR /src/wireguard-go

RUN go get -u github.com/wolfssl/go-wolfssl \
    && go mod edit -replace github.com/wolfssl/go-wolfssl=/src/go-wolfssl

# Build produces `wolfguard-go` (patch changes Makefile target)
RUN make

# --- runtime image ---
FROM debian:bookworm-slim AS runtime

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    wireguard-tools \
    iproute2 \
    socat \
    && rm -rf /var/lib/apt/lists/*

# wolfSSL shared library + loader cache
COPY --from=builder /usr/local/lib/libwolfssl.so* /usr/local/lib/
COPY --from=builder /etc/ld.so.cache /etc/ld.so.cache
COPY --from=builder /etc/ld.so.conf /etc/ld.so.conf
COPY --from=builder /etc/ld.so.conf.d /etc/ld.so.conf.d

# wolfguard-go binary
COPY --from=builder /src/wireguard-go/wolfguard-go /usr/local/bin/wolfguard-go

# Force userspace even when the kernel WireGuard module exists (otherwise wireguard-go exits).
ENV WG_I_PREFER_BUGGY_USERSPACE_TO_POLISHED_KMOD=1
ENV LD_LIBRARY_PATH="/usr/local/lib"

ENTRYPOINT ["/usr/local/bin/wolfguard-go"]
CMD ["-f", "wg0"]


