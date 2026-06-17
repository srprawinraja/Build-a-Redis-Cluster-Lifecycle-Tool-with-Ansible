# Redis Cluster Lifecycle Tool

A DevOps CLI tool wrapper for Ansible to provision, operate, scale, and perform zero-downtime rolling upgrades/rollbacks of a Redis cluster running inside Docker/Podman containers.

## Prerequisites

Before running the lifecycle tool, ensure your host machine (control node) has the following dependencies installed:

1. **Python 3:** Required to run the `redis-tool` CLI.
2. **Container Runtime:** Docker Engine (with Compose v2) or Podman (with `podman-compose`). (Podman is recommended for open-source rootless compliance).
3. **Ansible:** `ansible-playbook` version `2.14` or higher.
4. **SSH Key:** An SSH key pair generated on your host (e.g., `~/.ssh/id_rsa.pub`). If you don't have one, generate it with `ssh-keygen -t rsa -b 4096`.

### Automated Dependency Installation
If you are missing any required system dependencies (like Podman or Ansible), you can append the `--auto-install` flag to any command. The tool will check for missing dependencies and offer to install them automatically (asking for user confirmation before making changes):
```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1 --auto-install
```

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

### 6. Scale Out/In (Stretch Goals S1 & S2)
* **Scale Out:** Add a new master-replica pair (2 nodes) to the cluster and automatically rebalance the slots:
  ```bash
  ./redis-tool scale --add-nodes 2
  ```
* **Scale In:** Migrate all hash slots off a master node, and gracefully remove it and its replica from the cluster:
  ```bash
  ./redis-tool scale --remove-node <node-id>
  ```

### 7. Rollback (Stretch Goal S3)
If the upgrade needs to be undone or fails mid-way, you can rollback all upgraded nodes back to the previous version:
```bash
./redis-tool rollback --target-version 7.0.15
```

### 8. Idempotency (Stretch Goal S4)
All commands in `redis-tool` enforce strict idempotency:
* **Provision Idempotency:** Running `provision` on an already-configured, healthy cluster acts as a safe, non-destructive update (or config refresh without data loss).
* **Upgrade Idempotency:** Running `upgrade` to a target version when all nodes are already running that version will immediately identify the state, output a message, and exit cleanly as a no-op without running playbooks.

### 9. Structured Logging & Auditing (Stretch Goal S5)
Every lifecycle action executed by the tool is saved to the `logs/` directory in structured JSON format:
* **Centralized Operations Audit:** `logs/operations.json` contains a continuous list of all operations, including execution timestamps, target nodes, actions taken, and final outcomes.
* **Per-Run Details:** A discrete log is created for each individual command execution (e.g., `logs/op_YYYYMMDD_HHMMSS_<command>.json`).

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

## Rollback Strategy & Data Protection Guarantee

The rollback procedure ensures that the cluster can return to a previous version (e.g., 7.0.15) safely without data loss or compatibility conflicts:

1. **Deterministic Data Backup:** Connects to the master nodes and backs up all stored key-value pairs.
2. **Cluster Teardown & Version Cleanup:** Stops the Redis service across all nodes and wipes the old `/tmp/dump.rdb` and `/tmp/appendonlydir` data files to prevent RDB version conflicts (older Redis versions cannot parse RDB files created by newer versions).
3. **Downgrade Installation:** Re-installs and starts the older target version on all nodes.
4. **Data Restoration & Verification:** Restores the backed up key-value pairs to the newly initialized cluster and performs data integrity checks to verify all 1,000 keys are intact and correct.

---

## Design Choices & Assumptions

* **Ansible SSH Execution:** Direct cluster interaction from the host is bypassed. All commands are coordinated internally via Ansible SSH to resolve WSL container routing constraints.
* **Volume-Mounted Public Keys:** Utilizes volume-mounting of `id_rsa.pub` in `compose.yml` along with `StrictModes no` inside the container `Dockerfile` to handle key permissions seamlessly.
* **Pre-Baked Images:** Build tools (`build-essential` and `ca-certificates`) are baked directly into the Docker image, reducing upgrade runtimes by removing slow package downloads.
* **Idempotency Guards (Stretch Goal S4):** Running `provision` on an already-provisioned cluster is a safe no-op (updating configuration without losing data). Running `upgrade` to a target version when all nodes are already at that version will exit cleanly and log the check.
* **Structured Logging (Stretch Goal S5):** Every command writes a detailed JSON entry in the `logs/` directory (audit logs at `logs/operations.json` and individual command run files). Logs include timestamps, target nodes, actions taken, and output results.

---

## Known Limitations

1. **Host-to-Cluster Connectivity:** Redis cluster clients running on the host machine may not be able to reach Docker-internal cluster addresses (`10.10.0.x`). Therefore, cluster-aware operations and data seeding/verification are executed from within the container network or through Ansible automation.
2. **Dynamic Initial Sizing:** The initial cluster provisioning command assumes a standard six-node topology (3 masters and 3 replicas) and does not support dynamic topology sizing during the *initial* phase (though it can be scaled out/in post-provisioning).
3. **SSH Host Key Verification:** SSH host key verification has been disabled (`StrictHostKeyChecking=no`) for development and testing automation convenience in local virtual networks.
