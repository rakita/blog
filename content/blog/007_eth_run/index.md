+++
title = "Ethereum should be easier to run"
description = "It should be easy as `brew install eth` and `eth run`"
date = 2025-11-12T12:00:00+00:00
updated = 2025-11-12T12:00:00+00:00
draft = false
template = "blog/page.html"

[taxonomies]
authors = ["draganrakita"]
+++


# Ethereum should be easier to run

Running an Ethereum node is challenging. You need to understand what EL and CL are, how to set them up, and where to find them. The terminology around full nodes, history nodes, snap sync, and snapshot sync is confusing. Ethereum needs a single entry point that brings it all together in one place.

As complexity increases, new tools needs to remove that extra complexity and simplify the process.

This should be as easy as:
```bash
brew install eth
eth run
```

## `eth run`

The `eth run` command simplifies Ethereum node setup by automatically configuring and launching both Execution Layer (EL) and Consensus Layer (CL) clients with optimal settings based on your hardware.

It automatically handles genesis block initialization and supports multiple networks including mainnet, testnets (Sepolia, Holesky), and custom devnets, configuring the appropriate genesis and network parameters for each.

Before starting clients, it checks system resources (CPU, disk type and size, RAM, bandwidth) and provides recommendations if any requirements aren't met. It verifies port availability, detects existing running clients, checks firewall configuration, and validates data directory permissions.

The command automatically generates optimized configuration files for both EL and CL clients, setting appropriate cache sizes, database types, sync modes, and resource limits based on available hardware.

When executed, `eth run` starts both clients in the background, monitors their health, displays real-time sync progress, and integrates with `eth status` for detailed feedback. It handles errors gracefully—detecting crashes, port conflicts, network issues, and sync stalls—with automatic recovery attempts and clear remediation suggestions.

## `eth status`

Displays comprehensive status information about running Ethereum clients and infrastructure at a glance:

**Client & Sync Status:**
- EL/CL client status (running/syncing/stopped), versions, and uptime
- Current block number, sync progress percentage, and estimated time to full sync
- CL finalized epoch and head slot

**Network & Resources:**
- Connected peer count for both EL and CL
- Network type, chain ID, and health indicator
- CPU, memory, disk usage, and network bandwidth metrics

**APIs & Health:**
- RPC endpoint status and response time
- Overall health status with error messages and warnings
- Last successful block processed timestamp

For example:
```bash
$ eth status

Sync Status:
  EL: Block 18,500,000 / 18,500,000 (100%) ✓ Fully synced
  CL: Epoch 250,000 / 250,000 (100%) ✓ Fully synced
```


## `eth monitor`

The `eth monitor` command sets up a comprehensive monitoring stack for your Ethereum node, automatically deploying Prometheus, Grafana, and the ethPandaOps Metrics Exporter with zero configuration. It detects running EL and CL clients, configures metrics exporters accordingly, and loads pre-configured templated dashboards that adapt to your client setup.

```bash
$ eth monitor --start
[INFO] Starting monitoring stack...
[INFO] Prometheus starting on port 9090
[INFO] Grafana starting on port 3000

Monitoring stack running:
  - Grafana:     http://localhost:3000
  - Prometheus:  http://localhost:9090
  - Metrics API: http://localhost:9091/metrics
```

The stack collects comprehensive metrics from both execution and consensus layers (block processing, sync status, network, RPC performance, validator metrics) plus system resources. It provides multiple dashboard types (overview, execution client, consensus client, network, resources) and maybe even optional AlertManager integration with email, Slack, Discord, and PagerDuty. Configuratio. The command transforms complex monitoring setup into a single command providing observability.

## `eth dev`

The `eth dev` command launches a local development Ethereum client (for example, Foundry Anvil), optimized for smart contract development and testing. It can fork mainnet or testnets at any block height, provides pre-funded accounts, and executes transactions instantly with configurable block times.

```bash
$ eth dev --fork mainnet --block 18500000
[INFO] Forking mainnet at block 18500000
[INFO] ✓ Anvil started on http://localhost:8545
```

Supports multiple client dev modes and development environments (such as Foundry).

## `eth docker`

The `eth docker` command containerizes all functionality of `eth run`, `eth status`, `eth monitor`, and `eth infra` using Docker, providing consistent, isolated environments across different systems. It orchestrates multi-container setups with EL and CL clients, optional monitoring stack, and automatically generates a `docker-compose.yml` file. Data persists in Docker volumes across restarts.

```bash
$ eth docker --network mainnet --el-client geth --cl-client lighthouse --monitor
[INFO] Starting containers...
[INFO] ✓ geth container started (healthy)
[INFO] ✓ lighthouse container started (healthy)

Ethereum node running in Docker:
  - EL RPC: http://localhost:8545
  - CL API: http://localhost:5052
```

## `eth kurtosis`

The `eth kurtosis` command leverages [Kurtosis packages](https://github.com/ethpandaops/ethereum-package) to deploy complete Ethereum infrastructure stacks with a single command. It orchestrates multi-node networks, monitoring stacks, MEV infrastructure, and testing tools using portable, modular environments.

```bash
$ eth kurtosis --network devnet --nodes 4 --validators 64
[INFO] Deploying Ethereum package...
[INFO] Starting 4 nodes with 64 validators each...

Ethereum network running:
  - Network: Private devnet (Chain ID: 1337)
  - Nodes: 4 EL/CL pairs, 256 validators
  - EL RPC: http://localhost:8545
  - Grafana: http://localhost:3000
```

Supports advanced features like MEV-Boost (`--mev-type flashbots`), multi-client diversity, PeerDAS, monitoring, and block explorers. All services are automatically networked and configured, with easy cleanup via `--cleanup`.


## And more (testing, keys, validations, mev-boost, rpc calls)

One binary to rule them all, that sets up all configs and orchestrates all other binaries.