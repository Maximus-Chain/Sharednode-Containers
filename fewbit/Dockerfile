# ============================================
# FewBit Core - Production Dockerfile
# Multi-stage build for minimal image size
# ============================================

# syntax=docker/dockerfile:1.4

# ============================================
# Stage 1: Builder
# Build fewbitd from source
# ============================================
FROM ubuntu:jammy AS builder

# Install build dependencies
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  build-essential \
  libtool \
  autotools-dev \
  automake \
  pkg-config \
  bsdmainutils \
  python3 \
  bison \
  curl \
  ca-certificates \
  wget \
  git \
  autoconf \
  ccache \
  libboost-dev \
  libboost-filesystem-dev \
  libboost-system-dev \
  libboost-test-dev \
  libevent-dev \
  libminiupnpc-dev \
  libnatpmp-dev \
  libsqlite3-dev \
  libzmq3-dev \
  libgmp-dev \
  libsodium-dev \
  libssl-dev \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /home/fewbit

# Clone FewBit repository (main branch, shallow)
ARG FEWBIT_REPO=https://github.com/fewbit-network/Core-Wallet.git
ARG FEWBIT_BRANCH=main
RUN git clone --depth 1 --branch ${FEWBIT_BRANCH} ${FEWBIT_REPO} /home/fewbit/build

WORKDIR /home/fewbit/build

# Build depends system (boost, libevent, etc.) for the target host
ARG TARGETPLATFORM
RUN if [ "$TARGETPLATFORM" = "linux/arm64" ]; then \
  export DEPENDS_HOST=aarch64-linux-gnu; \
  else \
  export DEPENDS_HOST=x86_64-pc-linux-gnu; \
  fi && \
  make -C depends HOST=${DEPENDS_HOST} NO_UPNP=1 -j$(nproc)

# Configure and build fewbitd using the depends-built boost
ENV PATH="/usr/lib/ccache:$PATH"
ARG TARGETPLATFORM
RUN if [ "$TARGETPLATFORM" = "linux/arm64" ]; then \
  export DEPENDS_HOST=aarch64-linux-gnu; \
  else \
  export DEPENDS_HOST=x86_64-pc-linux-gnu; \
  fi && \
  ./autogen.sh && \
  ./configure --prefix=/home/fewbit/build/depends/${DEPENDS_HOST} \
  --bindir=/home/fewbit/build/out/bin \
  --libdir=/home/fewbit/build/out/lib \
  --with-gui=no \
  --disable-bench \
  --disable-tests \
  --disable-wallet \
  --enable-reduce-exports \
  --disable-dependency-tracking \
  --disable-shared \
  --enable-static \
  LDFLAGS="-static-libstdc++" && \
  make -j$(nproc) && \
  make install

# Create staging directory with binaries
RUN mkdir -p /build-output && \
  cp /home/fewbit/build/out/bin/fewbitd /home/fewbit/build/out/bin/fewbit-cli /home/fewbit/build/out/bin/fewbit-tx /build-output/ 2>/dev/null || \
  cp /home/fewbit/build/out/bin/fewbitd /home/fewbit/build/out/bin/fewbit-cli /build-output/; \
  strip /build-output/*

# ============================================
# Stage 2: Runtime
# Minimal image with only necessary binaries
# ============================================
FROM ubuntu:jammy AS runtime

LABEL maintainer="FewBit Developers <dev@fewbit.online>"
LABEL description="FewBit Core node daemon for blockchain operations"
LABEL org.opencontainers.image.source="https://github.com/fewbit-network/Core-Wallet"

# Security labels
USER root

# Create non-root user for running daemon
ARG USER_ID=1000
ARG GROUP_ID=1000

ENV USER_ID=${USER_ID:-1000}
ENV GROUP_ID=${GROUP_ID:-1000}
ENV HOME=/home/fewbit
ENV DATA_DIR=/home/fewbit/.fewbitcore

RUN groupadd -g ${GROUP_ID} fewbit && \
  useradd -u ${USER_ID} -g fewbit -s /bin/bash -m -d /home/fewbit fewbit && \
  mkdir -p ${DATA_DIR} && \
  chown ${USER_ID}:${GROUP_ID} -R /home/fewbit

# Install runtime dependencies only
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  ca-certificates \
  curl \
  unzip \
  libboost-system1.74.0 \
  libboost-filesystem1.74.0 \
  libevent-2.1-7 \
  libevent-pthreads-2.1-7 \
  libminiupnpc17 \
  libnatpmp1 \
  libsqlite3-0 \
  libzmq5 \
  libgmp10 \
  libsodium23 \
  && rm -rf /var/lib/apt/lists/* \
  && apt-get clean \
  && rm -rf /var/cache/apt/archives/*

# Copy binaries from builder
COPY --from=builder /build-output/* /usr/local/bin/

# Copy entrypoint script
COPY docker/entrypoint.sh /docker-entrypoint.sh
# Set permissions before switching to non-root user
RUN chmod +x /docker-entrypoint.sh

# Create required directories
RUN mkdir -p ${DATA_DIR} && \
  chown fewbit:fewbit ${DATA_DIR}

# Volume for persistent data
VOLUME ["${DATA_DIR}"]

# Expose P2P and RPC ports (mainnet + testnet)
# Mainnet: P2P=1155, RPC=1154
# Testnet: P2P=11551, RPC=11541
EXPOSE 1154 1155 11541 11551

# Set working directory
WORKDIR /home/fewbit

# Switch to non-root user
USER fewbit

# Default entrypoint
ENTRYPOINT ["/docker-entrypoint.sh"]
