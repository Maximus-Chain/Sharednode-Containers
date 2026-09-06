# ============================================
# Filopow Core - Production Dockerfile
# Multi-stage build for minimal image size
# ============================================

# syntax=docker/dockerfile:1.4

# ============================================
# Stage 1: Builder
# Build filopowd from source
# ============================================
FROM ubuntu:jammy AS builder

# Install build dependencies + system packages (boost, openssl, libevent, zmq,
# miniupnpc, gmp) that filopow links against. We do NOT run `make -C depends`
# (boost 1.70 from depends fails to build against gcc-11). Instead we use the
# system packages, and build only chia_bls separately (see below).
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
  cmake \
  libssl-dev \
  libboost-dev \
  libboost-filesystem-dev \
  libboost-system-dev \
  libboost-thread-dev \
  libboost-chrono-dev \
  libboost-test-dev \
  libboost-program-options-dev \
  libevent-dev \
  libminiupnpc-dev \
  libzmq3-dev \
  libgmp-dev \
  libdb-dev \
  libdb++-dev \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /home/filopow

# Clone Filopow repository (main branch, shallow, pinned to tag v5.2.0)
ARG FILOPOW_REPO=https://github.com/filoproject/filopow.git
ARG FILOPOW_BRANCH=main
ARG FILOPOW_TAG=v5.2.0
RUN git clone --depth 1 --branch ${FILOPOW_BRANCH} ${FILOPOW_REPO} /home/filopow/build && \
  cd /home/filopow/build && \
  git fetch --depth 1 origin tag ${FILOPOW_TAG} && \
  git checkout ${FILOPOW_TAG}

# Build and install chia_bls (Dash fork of Chia BLS signatures) to /usr/local.
# filopowd links against `-lchiabls -lgmp` via BLS_LIBS in src/Makefile.am, so we
# need libchiabls in the linker search path.
# Note: filopow's depends pins v20181101 which is broken on modern GCC due to
# relic's blake2.h using `#pragma pack(push, 1)` on structs containing uint32_t/
# uint64_t, leaving the blake2s_state/blake2b_state struct sizes unaligned.
# We patch the structs to add explicit padding so they are 8-byte multiples,
# which makes the array-of-state-strings pattern compile cleanly.
ARG CHIABLS_VERSION=v20181101
RUN git clone --depth 1 --branch ${CHIABLS_VERSION} \
    https://github.com/dashpay/bls-signatures.git /tmp/bls-signatures && \
  if [ -f /tmp/bls-signatures/contrib/relic/src/md/blake2.h ]; then \
    # Modern gcc rejects `blake2s_state S[8][1]` and `blake2b_state S[4][1]`
    # because the structs have ALIGNME(64) but their sizes (185, 361 bytes)
    # aren't multiples of 64. Drop the 64-byte alignment attribute on the
    # state structs (the param structs keep it because of pack(1)).
    sed -i 's|ALIGNME( 64 ) typedef struct __blake2s_state|typedef struct __blake2s_state|' \
      /tmp/bls-signatures/contrib/relic/src/md/blake2.h && \
    sed -i 's|ALIGNME( 64 ) typedef struct __blake2b_state|typedef struct __blake2b_state|' \
      /tmp/bls-signatures/contrib/relic/src/md/blake2.h; \
  fi && \
  # Drop the test executables (runtest/runbench) that pull in catch.hpp which
  # uses `sysconf()` in a constexpr context — rejected by gcc-11+. We only
  # need libchiabls.a.
  sed -i '/add_executable(runtest test.cpp)/,/endif()/{/add_executable(runtest/d;/add_executable(runbench/d;/target_link_libraries(runtest/d;/target_link_libraries(runbench/d}' \
    /tmp/bls-signatures/src/CMakeLists.txt && \
  mkdir -p /tmp/bls-signatures/build && cd /tmp/bls-signatures/build && \
  cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr/local \
    -DBUILD_ONLY_LIB=ON \
    -DSTLIB=ON \
    -DSHLIB=OFF \
    -DSTBIN=OFF && \
  make -j$(nproc) && \
  make install && \
  ldconfig

WORKDIR /home/filopow/build

# Replace all ax_boost_*.m4 files with simpler versions that don't have the
# buggy `break`-inside-AC_CHECK_LIB-action issue. The upstream macros all
# suffer from the same spurious "Could not link against ..." configure error
# on modern autoconf. We trust the system boost on Ubuntu 22.04 (libboost 1.74).
RUN cat > build-aux/m4/ax_boost_filesystem.m4 <<'M4EOF'
AC_DEFUN([AX_BOOST_FILESYSTEM],
[
    AC_DEFINE(HAVE_BOOST_FILESYSTEM,,[define if the Boost::Filesystem library is available])
    BOOST_FILESYSTEM_LIB="-lboost_filesystem"
    AC_SUBST(BOOST_FILESYSTEM_LIB)
])
M4EOF
RUN cat > build-aux/m4/ax_boost_system.m4 <<'M4EOF'
AC_DEFUN([AX_BOOST_SYSTEM],
[
    AC_DEFINE(HAVE_BOOST_SYSTEM,,[define if the Boost::System library is available])
    BOOST_SYSTEM_LIB="-lboost_system"
    AC_SUBST(BOOST_SYSTEM_LIB)
])
M4EOF
RUN cat > build-aux/m4/ax_boost_program_options.m4 <<'M4EOF'
AC_DEFUN([AX_BOOST_PROGRAM_OPTIONS],
[
    AC_DEFINE(HAVE_BOOST_PROGRAM_OPTIONS,,[define if the Boost::Program_Options library is available])
    BOOST_PROGRAM_OPTIONS_LIB="-lboost_program_options"
    AC_SUBST(BOOST_PROGRAM_OPTIONS_LIB)
])
M4EOF
RUN cat > build-aux/m4/ax_boost_thread.m4 <<'M4EOF'
AC_DEFUN([AX_BOOST_THREAD],
[
    AC_DEFINE(HAVE_BOOST_THREAD,,[define if the Boost::Thread library is available])
    BOOST_THREAD_LIB="-lboost_thread"
    AC_SUBST(BOOST_THREAD_LIB)
])
M4EOF
RUN cat > build-aux/m4/ax_boost_chrono.m4 <<'M4EOF'
AC_DEFUN([AX_BOOST_CHRONO],
[
    AC_DEFINE(HAVE_BOOST_CHRONO,,[define if the Boost::Chrono library is available])
    BOOST_CHRONO_LIB="-lboost_chrono"
    AC_SUBST(BOOST_CHRONO_LIB)
])
M4EOF

# Patch httpserver.cpp to include <deque> (missing in v5.2.0; std::deque is used
# in the WorkQueue template but the header is not included, causing build failure
# on modern gcc).
RUN sed -i 's|#include <condition_variable>|#include <condition_variable>\n#include <deque>|' src/httpserver.cpp && \
  grep -q '#include <deque>' src/httpserver.cpp

# Patch miner.cpp to guard wallet/wallet.h include. With --disable-wallet the
# Berkeley DB headers (db_cxx.h) are not available, but miner.cpp includes
# wallet/wallet.h unconditionally. All wallet usages in miner.cpp are already
# guarded with #ifdef ENABLE_WALLET, so the include just needs the same guard.
RUN sed -i 's|#include "wallet/wallet.h"|#ifdef ENABLE_WALLET\n#include "wallet/wallet.h"\n#endif|' src/miner.cpp && \
  grep -q '#ifdef ENABLE_WALLET' src/miner.cpp

# Patch validationinterface.cpp and validation.cpp to bring boost::placeholders
# into scope. Boost >= 1.73 removed the implicit `_1`/`_2`/`_3` placeholders
# from the global namespace; modern boost requires
# `using namespace boost::placeholders;` or explicit `boost::placeholders::_1`
# qualification.
RUN sed -i 's|#include <boost/signals2/signal.hpp>|#include <boost/signals2/signal.hpp>\n#include <boost/bind/placeholders.hpp>\nusing namespace boost::placeholders;|' src/validationinterface.cpp && \
  grep -q 'using namespace boost::placeholders' src/validationinterface.cpp && \
  sed -i 's|#include <boost/thread.hpp>|#include <boost/thread.hpp>\n#include <boost/bind/placeholders.hpp>\nusing namespace boost::placeholders;|' src/validation.cpp && \
  grep -q 'using namespace boost::placeholders' src/validation.cpp

# Configure and build filopowd against system packages and the chia_bls we just built.
ENV PATH="/usr/lib/ccache:$PATH"
RUN ./autogen.sh && \
  ./configure --prefix=/usr/local \
  --bindir=/usr/local/bin \
  --libdir=/usr/local/lib \
  --with-gui=no \
  --disable-bench \
  --disable-tests \
  --enable-reduce-exports \
  --disable-dependency-tracking \
  --with-incompatible-bdb \
  LDFLAGS="-L/usr/local/lib -static-libstdc++ -lstdc++fs" && \
  make -j$(nproc) && \
  make install

# Stage binaries into /build-output for the runtime image.
# Note: filopow does NOT build filopow-tx (it is commented out in src/Makefile.am,
# `bin_PROGRAMS += filopow-cli #filopow-tx`). Only filopowd + filopow-cli.
RUN mkdir -p /build-output && \
  cp /usr/local/bin/filopowd /usr/local/bin/filopow-cli /build-output/ && \
  strip /build-output/*

# ============================================
# Stage 2: Runtime
# Minimal image with only necessary binaries
# ============================================
FROM ubuntu:jammy AS runtime

LABEL maintainer="Filopow Developers <admin@github.com/filoproject/filopow>"
LABEL description="Filopow Core node daemon for blockchain operations"
LABEL org.opencontainers.image.source="https://github.com/filoproject/filopow"

# Security labels
USER root

# Create non-root user for running daemon
ARG USER_ID=1000
ARG GROUP_ID=1000

ENV USER_ID=${USER_ID:-1000}
ENV GROUP_ID=${GROUP_ID:-1000}
ENV HOME=/home/filopow
ENV DATA_DIR=/home/filopow/.filopowcore

RUN groupadd -g ${GROUP_ID} filopow && \
  useradd -u ${USER_ID} -g filopow -s /bin/bash -m -d /home/filopow filopow && \
  mkdir -p ${DATA_DIR} && \
  chown ${USER_ID}:${GROUP_ID} -R /home/filopow

# Install runtime dependencies only.
# libchiabls was statically linked into filopowd at build time, so we only need
# the runtime libs that filopowd links dynamically.
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  ca-certificates \
  curl \
  unzip \
  libboost-system1.74.0 \
  libboost-filesystem1.74.0 \
  libboost-program-options1.74.0 \
  libboost-thread1.74.0 \
  libboost-chrono1.74.0 \
  libevent-2.1-7 \
  libevent-pthreads-2.1-7 \
  libminiupnpc17 \
  libzmq5 \
  libgmp10 \
  libdb++5.3 \
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
  chown filopow:filopow ${DATA_DIR}

# Volume for persistent data
VOLUME ["${DATA_DIR}"]

# Expose P2P and RPC ports (mainnet + testnet)
# Mainnet: P2P=7767, RPC=7766
# Testnet: P2P=17767, RPC=17766
EXPOSE 7766 7767 17766 17767

# Set working directory
WORKDIR /home/filopow

# Switch to non-root user
USER filopow

# Default entrypoint
ENTRYPOINT ["/docker-entrypoint.sh"]
