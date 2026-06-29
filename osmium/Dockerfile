# ============================================
# Osmium Core - Production Dockerfile
# Multi-stage build for minimal image size
# ============================================

# syntax=docker/dockerfile:1.4

# ============================================
# Stage 1: Builder
# Build osmiumd from source
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

WORKDIR /home/osmium

# Clone Osmium repository (master branch, shallow)
ARG OSMIUM_REPO=https://github.com/osmium-labs/osmium.git
ARG OSMIUM_BRANCH=master
RUN git clone --depth 1 --branch ${OSMIUM_BRANCH} ${OSMIUM_REPO} /home/osmium/build

# Apply Osmium patch: lower testnet SupernodeHeight from 167176 to 100
# (target line is uniquely present in CTestNetParams only)
RUN sed -i 's/consensus\.SupernodeHeight = 167176;/consensus.SupernodeHeight = 100;/' \
  /home/osmium/build/src/chainparams.cpp && \
  grep -q 'consensus.SupernodeHeight = 100;' /home/osmium/build/src/chainparams.cpp

WORKDIR /home/osmium/build

# Build depends system (optional, skip if not needed)
ARG TARGETPLATFORM
ENV USE_BUILD_DEPENDS=1
RUN if [ "$TARGETPLATFORM" = "linux/arm64" ]; then \
  make HOST=aarch64-linux-gnu -j$(nproc) || true; \
  else \
  make HOST=x86_64-pc-linux-gnu -j$(nproc) || true; \
  fi || true

# Configure and build osmiumd
ENV PATH="/usr/lib/ccache:$PATH"
RUN if [ "$TARGETPLATFORM" = "linux/arm64" ]; then \
  ./autogen.sh && \
  ./configure --prefix=/usr/local \
  --enable-static \
  --disable-shared \
  --with-gui=no \
  --without-miniupnpc \
  --without-natpmp \
  --disable-bench \
  --disable-tests \
  --enable-reduce-exports \
  CXXFLAGS="-static-libstdc++"; \
  else \
  ./autogen.sh && \
  ./configure --prefix=/usr/local \
  --enable-static \
  --disable-shared \
  --with-gui=no \
  --without-miniupnpc \
  --without-natpmp \
  --disable-bench \
  --disable-tests \
  --enable-reduce-exports \
  CXXFLAGS="-static-libstdc++"; \
  fi

RUN make -j$(nproc)

# Create staging directory with binaries
RUN mkdir -p /build-output && \
  cp src/osmiumd src/osmium-cli src/osmium-tx /build-output/ 2>/dev/null || \
  cp src/osmiumd src/osmium-cli /build-output/; \
  strip /build-output/*

# ============================================
# Stage 2: Runtime
# Minimal image with only necessary binaries
# ============================================
FROM ubuntu:jammy AS runtime

LABEL maintainer="Osmium Developers <dev@osmium.space>"
LABEL description="Osmium Core node daemon for blockchain operations"
LABEL org.opencontainers.image.source="https://github.com/osmium-labs/osmium"

# Security labels
USER root

# Create non-root user for running daemon
ARG USER_ID=1000
ARG GROUP_ID=1000

ENV USER_ID=${USER_ID:-1000}
ENV GROUP_ID=${GROUP_ID:-1000}
ENV HOME=/home/osmium
ENV DATA_DIR=/home/osmium/.osmiumcore

RUN groupadd -g ${GROUP_ID} osmium && \
  useradd -u ${USER_ID} -g osmium -s /bin/bash -m -d /home/osmium osmium && \
  mkdir -p ${DATA_DIR} && \
  chown ${USER_ID}:${GROUP_ID} -R /home/osmium

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
  chown osmium:osmium ${DATA_DIR}

# Volume for persistent data
VOLUME ["${DATA_DIR}"]

# Expose P2P and RPC ports (mainnet + testnet)
# Mainnet: P2P=9969, RPC=9968
# Testnet: P2P=19969, RPC=19968
EXPOSE 9968 9969 19968 19969

# Set working directory
WORKDIR /home/osmium

# Switch to non-root user
USER osmium

# Default entrypoint
ENTRYPOINT ["/docker-entrypoint.sh"]
