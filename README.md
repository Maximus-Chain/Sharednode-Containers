# Sharednode-Containers

> Deployment configurations and CI pipelines for **Maximus SharedNode**-compatible blockchain node images. Dockerfiles, entrypoint scripts, Docker Compose stacks and GitHub Actions workflows for running nodes of multiple chains, with images published to GitHub Container Registry.

[![Build osmium](https://github.com/Maximus-Chain/Sharednode-Containers/actions/workflows/build-osmium.yml/badge.svg)](https://github.com/Maximus-Chain/Sharednode-Containers/actions/workflows/build-osmium.yml)
[![Build fewbit](https://github.com/Maximus-Chain/Sharednode-Containers/actions/workflows/build-fewbit.yml/badge.svg)](https://github.com/Maximus-Chain/Sharednode-Containers/actions/workflows/build-fewbit.yml)
[![Build filopow](https://github.com/Maximus-Chain/Sharednode-Containers/actions/workflows/build-filopow.yml/badge.svg)](https://github.com/Maximus-Chain/Sharednode-Containers/actions/workflows/build-filopow.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![GHCR](https://img.shields.io/badge/GHCR-maximus--chain-blue)](https://github.com/orgs/Maximus-Chain/packages)

## Overview

This repository packages blockchain node daemons derived from Bitcoin Core / Dash Core for deployment as minimal, production-ready Docker containers. Each chain directory is self-contained and ships:

- A multi-stage `Dockerfile` that fetches the chain source, compiles the daemon and produces a small runtime image running as a non-root user.
- An `entrypoint.sh` with snapshot bootstrap (`SNAPSHOT_URL`), graceful shutdown handling and `exec`-based startup.
- A `docker-compose.yml` for local development with mainnet, testnet and CLI profiles.

Two patterns are supported, depending on where the image is built and published.

## Chain Modes

### Mode A — Built-in chains

The chain source is fetched in the builder stage of our `Dockerfile`, compiled during CI and pushed to GitHub Container Registry by a workflow in `.github/workflows/build-<chain>.yml`. See [`osmium/`](./osmium) and [`fewbit/`](./fewbit) for working examples.

### Mode B — Referenced chains

The chain has its own upstream build pipeline. This repository only ships the orchestration files (Docker Compose) so users can pull the prebuilt image and run it the same standardized way. No `Dockerfile`, no `entrypoint.sh`, no GHCR workflow is added here for these chains.

## Supported Chains

### Built-in chains

Built and published from this repository.

| Chain   | Image                                              | Mainnet P2P / RPC | Testnet P2P / RPC | Workflow               |
|---------|----------------------------------------------------|-------------------|-------------------|------------------------|
| osmium  | `ghcr.io/maximus-chain/osmiumd`                    | 9969 / 9968       | 19969 / 19968     | `build-osmium.yml`     |
| fewbit  | `ghcr.io/maximus-chain/fewbitd`                    | 1155 / 1154       | 11551 / 11541     | `build-fewbit.yml`     |
| filopow | `ghcr.io/maximus-chain/filopowd`                   | 7767 / 7766       | 17767 / 17766     | `build-filopow.yml`    |
| _next_  | _planned_                                          | –                 | –                 | `build-<chain>.yml`    |

### Referenced chains

Orchestration only; built elsewhere.

| Chain    | Image                                            | Mainnet P2P / RPC | Testnet P2P / RPC | Upstream                       |
|----------|--------------------------------------------------|-------------------|-------------------|--------------------------------|
| maximus  | `ghcr.io/maximus-chain/maximusd`                 | 9939 / 9938       | 19939 / 19938     | `Maximus-Chain/maximus`        |
| _next_   | _planned_                                        | –                 | –                 | –                              |

## Quick Start

Pull a prebuilt image from GitHub Container Registry and run a mainnet node in seconds:

```bash
docker run -d --name osmium-mainnet \
  -p 9968:9968 -p 9969:9969 \
  -e DAEMON_ARGS="-rpcuser=osmium -rpcpassword=changeme_secure_password" \
  ghcr.io/maximus-chain/osmiumd:latest
```

For testnet, expose the testnet ports and add `-testnet=1` to `DAEMON_ARGS`:

```bash
docker run -d --name osmium-testnet \
  -p 19968:19968 -p 19969:19969 \
  -e DAEMON_ARGS="-testnet=1 -rpcuser=osmium -rpcpassword=changeme_secure_password" \
  ghcr.io/maximus-chain/osmiumd:latest
```

The same pattern works for any other built-in chain. For example, to run a FewBit mainnet node:

```bash
docker run -d --name fewbit-mainnet \
  -p 1154:1154 -p 1155:1155 \
  -e DAEMON_ARGS="-rpcuser=fewbit -rpcpassword=changeme_secure_password" \
  ghcr.io/maximus-chain/fewbitd:latest
```

And for FewBit testnet:

```bash
docker run -d --name fewbit-testnet \
  -p 11541:11541 -p 11551:11551 \
  -e DAEMON_ARGS="-testnet=1 -rpcuser=fewbit -rpcpassword=changeme_secure_password" \
  ghcr.io/maximus-chain/fewbitd:latest
```

The same pattern works for Filopow. For example, to run a Filopow mainnet node:

```bash
docker run -d --name filopow-mainnet \
  -p 7766:7766 -p 7767:7767 \
  -e DAEMON_ARGS="-rpcuser=filopow -rpcpassword=changeme_secure_password" \
  ghcr.io/maximus-chain/filopowd:latest
```

And for Filopow testnet:

```bash
docker run -d --name filopow-testnet \
  -p 17766:17766 -p 17767:17767 \
  -e DAEMON_ARGS="-testnet=1 -rpcuser=filopow -rpcpassword=changeme_secure_password" \
  ghcr.io/maximus-chain/filopowd:latest
```

> Note: Filopow's upstream `src/Makefile.am` only builds `filopowd` and `filopow-cli`; `filopow-tx` is commented out and is not shipped in the image.

## Environment Variables

The entrypoint accepts the following environment variables:

| Variable        | Default   | Description |
|-----------------|-----------|-------------|
| `DAEMON_ARGS`   | _(empty)_ | Extra command-line arguments passed to the daemon. Use this to set `-rpcuser`, `-rpcpassword`, `-testnet=1`, etc. |
| `SNAPSHOT_URL`  | _(empty)_ | URL of a `.tar.xz`, `.tar.gz` or `.zip` archive used to bootstrap the data directory on first run. Skipped automatically if `chainstate` already exists. |

A typical full setup with all variables used:

```bash
docker run -d --name osmium-mainnet \
  -p 9968:9968 -p 9969:9969 \
  -v osmium-data:/home/osmium/.osmiumcore \
  -e DAEMON_ARGS="-rpcuser=osmium -rpcpassword=changeme_secure_password -printtoconsole" \
  -e SNAPSHOT_URL="https://example.com/osmium-mainnet-snapshot.tar.xz" \
  ghcr.io/maximus-chain/osmiumd:latest
```

For FewBit, swap the image, user, data dir and ports:

```bash
docker run -d --name fewbit-mainnet \
  -p 1154:1154 -p 1155:1155 \
  -v fewbit-data:/home/fewbit/.fewbitcore \
  -e DAEMON_ARGS="-rpcuser=fewbit -rpcpassword=changeme_secure_password -printtoconsole" \
  -e SNAPSHOT_URL="https://example.com/fewbit-mainnet-snapshot.tar.xz" \
  ghcr.io/maximus-chain/fewbitd:latest
```

For Filopow:

```bash
docker run -d --name filopow-mainnet \
  -p 7766:7766 -p 7767:7767 \
  -v filopow-data:/home/filopow/.filopowcore \
  -e DAEMON_ARGS="-rpcuser=filopow -rpcpassword=changeme_secure_password -printtoconsole" \
  -e SNAPSHOT_URL="https://example.com/filopow-mainnet-snapshot.tar.xz" \
  ghcr.io/maximus-chain/filopowd:latest
```

## Local Development

Each chain ships a `docker-compose.yml` for local development and testing:

```bash
cd osmium
UID=$(id -u) GID=$(id -g) docker compose up -d

# Testnet node (under the `testnet` profile)
UID=$(id -u) GID=$(id -g) docker compose --profile testnet up -d

# One-shot debug CLI (under the `cli` profile, runs `osmium-cli getnetworkinfo`)
UID=$(id -u) GID=$(id -g) docker compose --profile cli run --rm osmium-cli
```

The same workflow works for FewBit:

```bash
cd fewbit
UID=$(id -u) GID=$(id -g) docker compose up -d
UID=$(id -u) GID=$(id -g) docker compose --profile testnet up -d
UID=$(id -u) GID=$(id -g) docker compose --profile cli run --rm fewbit-cli
```

And for Filopow:

```bash
cd filopow
UID=$(id -u) GID=$(id -g) docker compose up -d
UID=$(id -u) GID=$(id -g) docker compose --profile testnet up -d
UID=$(id -u) GID=$(id -g) docker compose --profile cli run --rm filopow-cli
```

> Note: Filopow's upstream `src/Makefile.am` only builds `filopowd` and `filopow-cli`; `filopow-tx` is commented out and is not shipped in the image.

The `UID`/`GID` exports match the build args so the daemon runs as your host user inside the container.

## Building from Source

Built-in chains can be built locally without any CI tooling:

```bash
git clone https://github.com/Maximus-Chain/Sharednode-Containers
cd Sharednode-Containers/osmium
docker build -t osmium-local .
```

Verify the binary:

```bash
docker run --rm --entrypoint /usr/local/bin/osmiumd osmium-local --version
# Osmium Core version v1.2.0
```

The same commands apply to FewBit:

```bash
cd Sharednode-Containers/fewbit
docker build -t fewbit-local .
docker run --rm --entrypoint /usr/local/bin/fewbitd fewbit-local --version
# FewBit Core version v6.6.0.0
```

And to Filopow:

```bash
cd Sharednode-Containers/filopow
docker build -t filopow-local .
docker run --rm --entrypoint /usr/local/bin/filopowd filopow-local --version
# FILOPOW Core Daemon version v5.2.0.0-d9a5b6a-dirty
```

> Note: Filopow's upstream `src/Makefile.am` only builds `filopowd` and `filopow-cli`; `filopow-tx` is commented out and is not shipped in the image.

## Adding a New Chain

Two options, depending on where the image is built.

### Option A — Built-in chain (we publish the image here)

1. Create `<chain>/Dockerfile` and `<chain>/docker/entrypoint.sh`, adapted from `osmium/` (rename user, home, data dir, ports, binary names).
2. Copy `osmium/docker/docker-compose.yml` to `<chain>/docker/`, updating service names, ports, volumes and image references.
3. Create `.github/workflows/build-<chain>.yml` from the `build-osmium.yml` template; replace the three inputs (`chain-name`, `context-path`, `image-name`).
4. (Optional) Apply upstream source patches during build with `RUN sed -i ...` in the Dockerfile's builder stage, validated with a `grep -q` so the build fails loudly if the patch did not apply.

### Option B — Referenced chain (image comes from upstream)

1. Create `<chain>/docker/docker-compose.yml` that references the upstream image, e.g. `image: ghcr.io/maximus-chain/<chain>d:1.2.0`.
2. Add mainnet, optionally testnet and CLI services, mirroring the structure used in `osmium/docker/docker-compose.yml`.
3. No `Dockerfile`, no `entrypoint.sh`, no GitHub Actions workflow is needed in this repo — image publishing happens in the upstream project.

## CI/CD & Publishing

The continuous integration pipeline is split into a single reusable workflow and one thin caller per built-in chain:

- `.github/workflows/build-chain.yml` — reusable workflow that performs the `docker buildx build` against `linux/amd64`, builds the tag list dynamically, pushes to `ghcr.io/maximus-chain/<chain>d` and verifies the resulting image with `<chain>d --version`.
- `.github/workflows/build-<chain>.yml` — caller; one per built-in chain, around 15 lines, only sets the three inputs (`chain-name`, `context-path`, `image-name`) and forwards to the reusable.

#### Triggers (per caller)

- **Manual**: `workflow_dispatch` from the Actions UI.
- **Automatic**: push to `main`, `master` or `develop` with paths under `<chain>/**`.
- **Release**: tag pushes matching `<chain>-*` (for example `osmium-v1.2.0`, `fewbit-v6.6.0.0` or `filopow-v5.2.0`).

#### Tags published

The reusable workflow computes the tag list dynamically based on the triggering event:

| Trigger                       | Tags published                                              |
|-------------------------------|-------------------------------------------------------------|
| Push to `main` / `master`     | `sha-<7>`, `latest`                                         |
| Push to `develop`             | `sha-<7>`, `develop`                                        |
| Tag `<chain>-v<version>`      | `sha-<7>`, `<chain>-v<version>`, `v<version>`, `latest`     |

The reusable workflow needs `permissions: packages: write` and `contents: read` on the caller, and per-chain Docker layer caches reduce repeat-build times.

Referenced chains are not built by CI in this repository; their publishing happens in their upstream project.

## Contributing

Issues and pull requests are welcome. New chains should follow the patterns in the **Adding a New Chain** section above. Before opening a PR, please build the affected chain locally and confirm the resulting binary prints `<chain>d` and its `--version` correctly.

## License

MIT — see [LICENSE](./LICENSE).
