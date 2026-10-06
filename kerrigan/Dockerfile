# ============================================
# Kerrigan Core - Production Dockerfile
# Multi-stage build for minimal image size
# ============================================

# syntax=docker/dockerfile:1.4

# ============================================
# Stage 1: Builder
# Build kerrigand from source
# ============================================
# Kerrigan builds against Ubuntu 24.04 system libraries (boost 1.83) and
# needs a Rust toolchain for its Sapling / HMP crates, so unlike the other
# built-in chains it uses noble and skips `make -C depends`.
FROM ubuntu:noble AS builder

ENV DEBIAN_FRONTEND=noninteractive

# Install build dependencies
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  build-essential \
  libtool \
  autotools-dev \
  automake \
  autoconf \
  pkg-config \
  bsdmainutils \
  python3 \
  cmake \
  curl \
  ca-certificates \
  git \
  ccache \
  libboost-dev \
  libboost-filesystem-dev \
  libboost-thread-dev \
  libevent-dev \
  libminiupnpc-dev \
  libnatpmp-dev \
  libzmq3-dev \
  libgmp-dev \
  libsodium-dev \
  && rm -rf /var/lib/apt/lists/*

# Rust toolchain pinned to upstream's rust-toolchain.toml, plus the
# cxxbridge CLI matching the `cxx` crate version in upstream Cargo.toml.
ENV RUSTUP_HOME=/usr/local/rustup \
  CARGO_HOME=/usr/local/cargo \
  PATH="/usr/local/cargo/bin:${PATH}"
ARG RUST_VERSION=1.81.0
ARG CXXBRIDGE_VERSION=1.0.186
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
  | sh -s -- -y --default-toolchain ${RUST_VERSION} --profile minimal && \
  cargo install cxxbridge-cmd --version ${CXXBRIDGE_VERSION} --locked

WORKDIR /home/kerrigan

# Clone Kerrigan repository (pinned release tag, shallow)
ARG KERRIGAN_REPO=https://github.com/kerrigan-network/kerrigan.git
ARG KERRIGAN_BRANCH=v1.3.2
RUN git clone --depth 1 --branch ${KERRIGAN_BRANCH} ${KERRIGAN_REPO} /home/kerrigan/build

WORKDIR /home/kerrigan/build

# Configure and build kerrigand against system libraries
ENV PATH="/usr/lib/ccache:$PATH"
RUN ./autogen.sh && \
  ./configure --prefix=/home/kerrigan/build/out \
  --with-gui=no \
  --disable-bench \
  --disable-tests \
  --disable-wallet \
  --disable-man \
  --enable-reduce-exports \
  --disable-dependency-tracking && \
  make -j$(nproc) && \
  make install

# Create staging directory with binaries
RUN mkdir -p /build-output && \
  cp /home/kerrigan/build/out/bin/kerrigand /home/kerrigan/build/out/bin/kerrigan-cli /home/kerrigan/build/out/bin/kerrigan-tx /build-output/ 2>/dev/null || \
  cp /home/kerrigan/build/out/bin/kerrigand /home/kerrigan/build/out/bin/kerrigan-cli /build-output/; \
  strip /build-output/*

# ============================================
# Stage 2: Runtime
# Minimal image with only necessary binaries
# ============================================
FROM ubuntu:noble AS runtime

LABEL maintainer="Maximus Chain <dev@maximuschain.com>"
LABEL description="Kerrigan Core node daemon for blockchain operations"
LABEL org.opencontainers.image.source="https://github.com/kerrigan-network/kerrigan"

ENV DEBIAN_FRONTEND=noninteractive

# Security labels
USER root

# Create non-root user for running daemon
ARG USER_ID=1000
ARG GROUP_ID=1000

ENV USER_ID=${USER_ID:-1000}
ENV GROUP_ID=${GROUP_ID:-1000}
ENV HOME=/home/kerrigan
ENV DATA_DIR=/home/kerrigan/.kerrigan

# ubuntu:noble ships a default `ubuntu` user with UID/GID 1000; drop it so
# the kerrigan user can take the requested IDs.
RUN (userdel -r ubuntu 2>/dev/null || true) && \
  groupadd -g ${GROUP_ID} kerrigan && \
  useradd -u ${USER_ID} -g kerrigan -s /bin/bash -m -d /home/kerrigan kerrigan && \
  mkdir -p ${DATA_DIR} && \
  chown ${USER_ID}:${GROUP_ID} -R /home/kerrigan

# Install runtime dependencies only
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  ca-certificates \
  curl \
  unzip \
  xz-utils \
  libboost-filesystem1.83.0 \
  libboost-thread1.83.0 \
  libevent-2.1-7t64 \
  libevent-pthreads-2.1-7t64 \
  libminiupnpc17 \
  libnatpmp1t64 \
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
  chown kerrigan:kerrigan ${DATA_DIR}

# Volume for persistent data
VOLUME ["${DATA_DIR}"]

# Expose P2P and RPC ports (mainnet + testnet)
# Mainnet: P2P=7120, RPC=7121
# Testnet: P2P=17120, RPC=17121
EXPOSE 7120 7121 17120 17121

# Set working directory
WORKDIR /home/kerrigan

# Switch to non-root user
USER kerrigan

# Default entrypoint
ENTRYPOINT ["/docker-entrypoint.sh"]
