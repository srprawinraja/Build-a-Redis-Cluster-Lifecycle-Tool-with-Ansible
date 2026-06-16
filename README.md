# Redis Cluster Lifecycle Tool

A DevOps CLI tool wrapper for Ansible to provision, operate, and perform zero-downtime rolling upgrades of a 6-node Redis cluster running inside Docker/Podman containers.

## Prerequisites

Before running the lifecycle tool, ensure your host machine (control node) has the following dependencies installed:

1. **Python 3:** Required to run the `redis-tool` CLI.
2. **Container Runtime:** Docker Engine (with Compose v2) or Podman (with `podman-compose`).
3. **Ansible:** `ansible-playbook` version `2.14` or higher.
4. **SSH Key:** An SSH key pair generated on your host (e.g., `~/.ssh/id_rsa.pub`). If you don't have one, generate it with `ssh-keygen -t rsa -b 4096`.

## Infrastructure Setup

To spin up the containerized network environment (representing 6 servers with static IPs):

1. **Rebuild the image with pre-installed build tools:**
   * Using **Docker**:
     ```bash
     cd infra
     docker build -t redis-node .
     ```
   * Using **Podman**:
     ```bash
     cd infra
     podman build -t redis-node .
     ```

2. **Bring up the cluster network:**
   * Using **Docker Compose**:
     ```bash
     docker compose up -d
     ```
   * Using **Podman Compose**:
     ```bash
     podman-compose up -d
     ```
   *This automatically volume-mounts your SSH public key (`~/.ssh/id_rsa.pub` or `~/.ssh/id_active`) into all containers for passwordless SSH access.*

---

## Usage Instructions

Run the `redis-tool` script from the project root:

### 1. Provision a Cluster (Phase 1)
```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```
*Installs Redis v7.0.15, configures cluster mode, starts servers, and joins them into a 3-master + 3-replica cluster.*

### 2. Seed and Verify Data (Phase 2)
```bash
./redis-tool data seed --keys 1000
./redis-tool data verify
```
*Seeds 1000 key-value pairs deterministically (values are SHA256 hashes of the keys) and verifies data integrity across all nodes.*

### 3. Check Cluster Status (Phase 3)
```bash
./redis-tool status
```
*Displays IP, port, role, slot ranges, key distribution, and memory usage per node.*

### 4. Perform Rolling Upgrade (Phase 4)
```bash
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
```
*Runs the zero-downtime rolling upgrade to Redis 7.2.6, verifying data integrity before and after the process.*

### 5. Run Full Verification (Phase 5)
```bash
./redis-tool verify --full
```
*Runs a comprehensive check on data integrity, version consistency, topology health, cluster state, and replication lag.*

---

## Upgrade Strategy & Zero-Downtime Guarantee

The rolling upgrade achieves **zero client-visible downtime** and keeps the cluster state `ok` at all times:

1. **Pre-Flight Validation:** Asserts cluster health, checks version differences, and verifies the baseline data integrity.
2. **Upgrading Replicas First:** Replicas do not serve slot traffic directly, so they are upgraded sequentially (one-by-one). The playbook stops Redis, installs version 7.2.6, restarts the node, and waits for replication sync to complete before proceeding.
3. **Upgrading Masters via Failover:** To avoid downtime on active masters:
   * A manual failover (`CLUSTER FAILOVER`) is triggered on its replica node.
   * The replica is promoted to master, and the original master is demoted to replica.
   * The demoted original master is then safely stopped, upgraded, and rejoined as a replica.
4. **Post-Upgrade Checks:** Verifies version consistency and checks data integrity.

---

## Design Choices & Assumptions

* **Ansible SSH Execution:** Direct cluster interaction from the host is bypassed. All commands are coordinated internally via Ansible SSH to resolve WSL container routing constraints.
* **Volume-Mounted Public Keys:** Utilizes volume-mounting of `id_rsa.pub` in `compose.yml` along with `StrictModes no` inside the container `Dockerfile` to handle key permissions seamlessly.
* **Pre-Baked Images:** Build tools (`build-essential` and `ca-certificates`) are baked directly into the Docker image, reducing upgrade runtimes by removing slow package downloads.

---

## Known Limitations

1. **Host-to-Cluster Connectivity:** Redis cluster clients running on the host machine may not be able to reach Docker-internal cluster addresses (`10.10.0.x`). Therefore, cluster-aware operations and data seeding/verification are executed from within the container network or through Ansible automation.
2. **Fixed Topology Size:** The current lifecycle automation assumes a fixed six-node topology (3 masters and 3 replicas) and does not support dynamic node sizing during initial provisioning.
3. **SSH Host Key Verification:** SSH host key verification has been disabled (`StrictHostKeyChecking=no`) for development and testing automation convenience in local virtual networks.

