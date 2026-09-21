# Run a Node

## Introduction

zkLink Nova self-hosted RPC node is based on zkSync external node. You can find detailed information about zkSync external node [here](https://docs.zksync.io/zksync-node). This section focus on how to build a zkLink Nova self-hosted RPC node.

## Quick Start

### Preferred hardware configuration

This configuration is approximate, expect updates to these specs.

* **Architecture**: AMD64
* **CPU**: 32 core
* **RAM**: 64 GB
* **Storage**:
  * Testnet - \~1 TB (at the time of writing) and will grow over time, so should be constantly monitored
  * Mainnet - \~2 TB (at the time of writing) and will grow over time, so should be constantly monitored
  * NVMe recommended
* **Network**: 100 Mbps network connection.

{% hint style="info" %}
A mainnet bring-up was measured end-to-end on 16 vCPU / 128 GB RAM / local NVMe. The CPU and RAM figures above are comfortable rather than mandatory — fewer cores mainly make the initial restore and Merkle tree rebuild take longer. Storage is the figure to respect: the restore alone consumes about 330 GB before the Merkle tree starts growing.
{% endhint %}

### Software Prerequisites

You can follow Docker's official [manuals](https://docs.docker.com/engine/install/) to install `docker compose` and `Docker`.

Use `docker compose` (Compose V2). The commands below are written for it.

### The mainnet snapshot

A self-hosted node is bootstrapped by restoring a PostgreSQL dump of the mainnet node's database. Download it here:

| | |
| - | - |
| Snapshot | [zklink\_nova\_en\_snapshot.backup](https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/zklink_nova_en_snapshot.backup) |
| SHA256 | [zklink\_nova\_en\_snapshot.backup.sha256](https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/zklink_nova_en_snapshot.backup.sha256) |
| Manifest | [MANIFEST.txt](https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/MANIFEST.txt) |
| Size | 42.3 GiB (45,439,961,311 bytes) |
| Content point | L2 block 8,354,475 · L1 batch 113,984 · 2026-09-16 11:30:03 UTC |
| Protocol version | 21 |

{% hint style="warning" %}
**Restore with PostgreSQL 14.** The archive is written by `pg_dump` 14, and `pg_restore` 14 cannot read archives produced by newer versions. The docker-compose file pins `postgres:14` for this reason — do not substitute a newer image.
{% endhint %}

The snapshot ships every table's structure, so the schema is complete and the node runs normally, but two groups of tables are shipped empty to keep the download manageable:

* `call_traces` — read only by the `debug` namespace (`debug_traceTransaction`, `debug_traceBlockByNumber`, `debug_traceBlockByHash`). Those methods return nothing for transactions older than the snapshot's content point. Your node records its own traces for every block it executes after startup, as long as `debug` stays in `EN_API_NAMESPACES`, so trace coverage is complete from your start block onward. `debug_traceCall` is unaffected — it replays in the VM and never reads the table.
* The prover pipeline tables (`prover_jobs_fri`, the witness and aggregation job tables, `proof_compression_jobs_fri`, `gpu_prover_queue_fri`, `proof_generation_details` and friends) — an external node never reads them.

### Running zkLink Nova self-hosted RPC Node locally

We offer a [docker-compose file](https://github.com/zkLinkProtocol/zksync-era/tree/zklink/docs/guides/external-node/docker-compose-examples) to facilitate running a zkLink Nova self-hosted rpc node locally.

#### Create data folder

```
sudo mkdir -p /data/mainnet-postgres
sudo mkdir -p /data/mainnet-rocksdb
sudo mkdir -p /data/mainnet-prometheus-data
sudo mkdir -p /data/mainnet-grafana-data
sudo mkdir -p /data/prometheus
sudo touch /data/prometheus/prometheus.yml
```

#### Fetch the compose file and adjust two settings

```
cd /data && wget https://github.com/zkLinkProtocol/zksync-era/raw/zklink/docs/guides/external-node/docker-compose-examples/docker-compose.yml
```

Before starting anything, add a file-descriptor limit and a restart policy to the `external-node` service:

{% code title="docker-compose.yml" overflow="wrap" lineNumbers="false" %}

```yaml
  external-node:
    image: "zklinkprotocol/nova-external-node:v1.2"
    restart: unless-stopped
    ulimits:
      nofile:
        soft: 1048576
        hard: 1048576
```

{% endcode %}

{% hint style="danger" %}
**Do not skip the `nofile` limit.** The Merkle tree's RocksDB accumulates one `.sst` file per compacted range as it grows. Against the mainnet snapshot it passes Docker's default soft limit of 1024 open files partway through the rebuild, and the node dies with:

`Failed writing a batch to RocksDB: IO error: While open a file for random read: ./db/ext-node/lightweight/NNNNNN.sst: Too many open files`

The process then exits with status **0**, so without `restart: unless-stopped` Docker will not bring it back and the node stays silently dead. This was reproduced at batch 27,840 of 113,985 with 991 `.sst` files open.
{% endhint %}

While editing, you may also delete the `command: --enable-snapshots-recovery` line. The node binary does not accept that argument and the image entrypoint discards container arguments, so it has no effect; it only misleads. Change `POSTGRES_PASSWORD=changeme` too if the host is not fully private.

#### start pg sevice

```
docker compose -f docker-compose.yml up -d postgres
docker compose -f docker-compose.yml ps
```

Note the Postgres container name from `ps`. With the compose file in `/data` it is `data-postgres-1`. The server listens on port **5430**, not 5432.

#### Import backup data from Snapshot

Start Postgres first, as above, and only then download the snapshot into its data directory — the directory has to be empty when Postgres initialises it.

{% code title="Shell" overflow="wrap" lineNumbers="true" fullWidth="false" %}

```markup
sudo curl -fL --retry 3 -C - -o /data/mainnet-postgres/zklink_nova_en_snapshot.backup https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/zklink_nova_en_snapshot.backup
curl -fsSL -O https://zklink-nova-en-snapshots.s3.ap-northeast-1.amazonaws.com/zklink_nova_en_snapshot.backup.sha256
cd /data/mainnet-postgres && sudo sha256sum -c ~/zklink_nova_en_snapshot.backup.sha256
docker exec -i data-postgres-1 psql -U postgres -p 5430 -c "create database zklink_ext_node"
docker exec -i data-postgres-1 pg_restore -O -U postgres -p 5430 -d zklink_ext_node -j $(nproc) /var/lib/postgresql/data/zklink_nova_en_snapshot.backup
```

{% endcode %}

Three details in those commands are easy to lose:

* **`sudo` on the download.** Once Postgres has initialised `/data/mainnet-postgres`, the directory belongs to the container's `postgres` user and an unprivileged `wget` there fails with permission denied.
* **`-C -` on the download.** The object supports range requests, so this resumes a broken transfer instead of restarting 42 GB.
* **`-O` on `pg_restore`.** Without it, `pg_restore` attempts 56 `ALTER TABLE … OWNER TO zklink` statements, fails every one with `role "zklink" does not exist`, prints `warning: errors ignored on restore: 56` and **exits non-zero**. The data restores correctly either way — the tables end up owned by the restoring user, which is what you want — but a non-zero exit will abort any wrapper script using `set -e`.

Check the restore before going further. A healthy result is 42 tables with no invalid indexes, and a last L2 block equal to the snapshot's content point:

{% code title="Shell" overflow="wrap" %}

```markup
docker exec -i data-postgres-1 psql -U postgres -p 5430 -d zklink_ext_node -c "SELECT pg_size_pretty(pg_database_size('zklink_ext_node')) AS size, (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public' AND c.relkind='r') AS tables, (SELECT count(*) FROM pg_index WHERE NOT indisvalid) AS invalid_indexes, (SELECT max(number) FROM miniblocks) AS last_l2_block"
```

{% endcode %}

Once the restore is verified you can delete the `.backup` file from the data directory to reclaim 42 GB.

#### Start zkLink Nova Self-hosted RPC Node

To start a mainnet instance, run:

```
docker compose -f docker-compose.yml up -d
```

To reset its state, run:

```
docker compose -f docker-compose.yml down --volumes
```

You can see the status of the node (after recovery) in the local Grafana dashboard. Those commands start external node locally inside docker. The HTTP JSON-RPC API can be accessed on port 3060 and WebSocket API can be accessed on port 3061. A healthcheck is served on port 3081.

The image entrypoint runs `sqlx database setup` before the node starts. That compares the migration checksums recorded in the snapshot against the migrations baked into the image and refuses to start on any mismatch, so a clean start is itself evidence that snapshot and image agree.

Within a minute the healthcheck should report `ready`:

```
curl -s http://127.0.0.1:3081/health
```

{% hint style="info" %}
Tips

After importing the data, the blocks will be scanned to restore the Merkle tree. Block synchronization will continue only after the recovery is complete, so a block number that does not advance during this phase is expected rather than a fault — confirm `next_l1_batch_number` is climbing in `/health` before investigating anything else.

**Measured on mainnet, 2026-09**

* Snapshot download: \~70 MB/s, about 10 minutes on a well-connected host. On a 100 Mbps link, budget an hour.
* Merkle tree rebuild: the rate **decays as the tree deepens** — do not extrapolate from the first hour. Measured \~2,640 batches/hour over the first \~1,000 batches (120 thousand leaves), down to \~846 batches/hour by batch 27,900 (8.0 million leaves).
* Batches to process: 113,985. At the rates above, budget **several days**, not the 5 hours quoted in earlier revisions of this page — that figure dates from when the chain was near 10,000 batches, and it scales worse than linearly with chain length.

**Get local latest block**

curl <http://localhost:3060> -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth\_blockNumber","params":\[],"id":1}'
{% endhint %}

{% hint style="success" %}
**The rebuild is a verification, not just a warm-up.** For every batch the node recomputes the state root from the restored data and compares it against the root hash the snapshot already carries — root hash, Merkle root, parent hash and L2→L1 log root, chained batch to batch. A mismatch is a hard failure logged as `Root hash verification failed`, so tree progress doubles as a running proof that your copy of the snapshot is sound.

`Root hash verification failed` is the one string worth alerting on. Do not alert on `reorg` — the reorg detector logs `Checking root hash match for earliest L1 batch #0` on every pass as normal behaviour, so that pattern only produces false alarms.
{% endhint %}

### Troubleshooting

| Symptom | Cause |
| - | - |
| `404 NoSuchBucket` on download | An older revision of this page pointed at an `ap-east-1` bucket that no longer exists. Use the URL in [The mainnet snapshot](#the-mainnet-snapshot). |
| Permission denied writing the snapshot | The Postgres container owns `/data/mainnet-postgres`. Use `sudo`. |
| `errors ignored on restore: 56` | Expected without `-O`; all 56 are `ALTER TABLE … OWNER TO zklink`. The restore is correct but the exit code is non-zero. |
| `unsupported version … in file header` | Restoring with a `pg_restore` older than the archive. Use PostgreSQL 14. |
| initdb fails, directory not empty | The snapshot was placed in the data directory before Postgres first started. Clear the directory, start Postgres, then download. |
| `Too many open files` then the node exits with status 0 | The `nofile` ulimit was not raised. See the compose edit above; the node resumes from where it stopped once restarted. |
| Node exits immediately at startup | `sqlx database setup` rejected a migration checksum — snapshot and image versions disagree. Check the log for the migration name. |
| Block number not advancing | Normal while the Merkle tree rebuilds. |
| `Root hash verification failed` | The recomputed state root disagrees with the snapshot's. Re-check the SHA256 and restore again from a fresh download. |

### Redeploying against a newer snapshot

Stop the node, remove the RocksDB directories (`/data/mainnet-rocksdb`), drop the database, restore the new snapshot, and start again. Reusing a stale RocksDB tree against a newer database is the one combination that will not recover on its own.

### How this guide was verified

Every step above was executed on a clean host in a different AWS region from the one serving the snapshot: anonymous download over the public URL, SHA256 checked byte-exact against the published checksum, `pg_restore` (42 tables, 105 indexes, 0 invalid indexes), the image's `sqlx database setup` accepting the snapshot's migration checksums, and the node starting and answering `eth_blockNumber` from the restored state.

The per-batch root-hash re-verification ran to **28,260 of 113,985 batches (24.8%)** with zero root-hash failures, zero reorg warnings and zero errors, and was then stopped deliberately — the rebuild rate decays as the tree deepens, and the archive's SHA256 already matches byte-for-byte. The single interruption along the way was the missing `nofile` limit described above: an environment fault, not a defect in the snapshot.


## Building External Node from Source Code

This document outlines how to build the External Node from the zkLink Nova source code, supporting three ways:

1. Building the binary file for the External Node directly from the source code.
2. Building the Docker image for the External Node directly from the source code.
3. Customizing the Docker image for the External Node built from the source code.

### Install Dependencies

Run the following commands to install the necessary dependencies.

{% hint style="info" %}
Note: If you only wish to build the Docker image for the zkLink Nova External Node from the source code, you only need to install the Docker-related dependencies.
{% endhint %}

#### Rust and Tools

```
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# SQL tools
cargo install sqlx-cli --version 0.7.3
```

#### NVM and Node/Yarn

```
# NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash

# Node & Yarn
nvm install 18
npm install -g yarn
yarn set version 1.22.19
```

#### System Software

```
# All necessary packages
sudo apt-get update
sudo apt-get install build-essential pkg-config cmake clang lldb lld libssl-dev postgresql ca-certificates curl
```

#### Docker

```
# Docker installation
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-compose docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker <YOUR_USER>
```

### Pulling Source Code

Use the command below to pull the zkLink Nova source code and fetch the submodules. If you wish to build code related to the zkLink Nova Sepolia testnet, please use the zklink\_testnet branch.

```
git clone -b zklink https://github.com/zkLinkProtocol/zksync-era.git
cd zksync-era
git submodule update --init --recursive
# Initialize build system
zk
# Build contract artifacts
zk compiler system-contracts
zk contract build
```

Run the command below to set up the working directory:

```
echo -e "export ZKSYNC_HOME=$(pwd)\nexport PATH=\$ZKSYNC_HOME/bin:\$PATH" >> ~/.bashrc && source ~/.bashrc
```

### Build the External Node

#### If you only wish to build the binary file for the External Node

Execute the command below; the results will be located in the target/release directory.

```
cargo build --release
```

#### If you want to build the Docker image for the External Node from source

Run the command below:

```
docker build -f docker/external-node/Dockerfile . -t <image name>:<tag>
```

#### If you want to customize the Docker image for the External Node

Refer to all the <mark style="background-color:green;">BUILD</mark> and <mark style="background-color:green;">COPY</mark> commands in the <mark style="background-color:green;">docker/external-node/Dockerfile</mark> file to customize the Docker image for the zkLink Nova External Node.
