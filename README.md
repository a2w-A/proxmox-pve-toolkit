# Proxmox PVE Toolkit (`proxmox-pve-toolkit`)

[![Proxmox VE 7.x & 8.x](https://img.shields.io/badge/Proxmox%20VE-7.x%20%7C%208.x-E57000?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%20%7C%203.11%20%7C%203.12-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![Docker Ready](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![Maintenance Status](https://img.shields.io/badge/Maintained%3F-actively%20developed-brightgreen.svg?style=for-the-badge)](https://github.com/algo2world/proxmox-pve-toolkit)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)

> **Enterprise-grade, modular Python CLI toolkit and automation engine for Proxmox Virtual Environment (PVE) 7.x and 8.x hypervisors.**  
> Built by **Algo2World & Ind. Ecosystem** with `typer`, `rich`, and `proxmoxer` for resilient infrastructure auditing, automated vzdump/snapshot retention orchestration, bulk workload power state management, and baseline security hardening.

---

## 1. Architectural Overview:

```
                      +---------------------------------------+
                      |         proxmox-pve-toolkit           |
                      |   (Typer CLI & Rich Terminal Engine)  |
                      +-------------------+-------------------+
                                          |
                +-------------------------+-------------------------+
                |                         |                         |
                v                         v                         v
       +-----------------+       +-----------------+       +-----------------+
       |  Node & Cluster |       |  VZDump Backup  |       | Bulk Power State|
       |  Health Auditor |       |  Orchestrator   |       |   Controller    |
       +--------+--------+       +--------+--------+       +--------+--------+
                |                         |                         |
                +-------------------------+-------------------------+
                                          |
                                          v
                               +--------------------+
                               | Security Baseline  |
                               | & Hardening Engine |
                               +----------+---------+
                                          |
                        HTTPS / REST API (Port 8006)
                     PVEAPIToken Auth / Scoped ACL Roles
                                          |
                                          v
           +-------------------------------------------------------------+
           |                 Proxmox VE Cluster / Standalone              |
           |   [ Node 1: pve-01 ]       [ Node 2: pve-02 ]     ...       |
           |   - QEMU Virtual Machines  - LXC Containers                 |
           |   - ZFS / Ceph / LVM-Thin  - Corosync Cluster & Firewall    |
           +-------------------------------------------------------------+
```

---

## 2. Core Capabilities

| Module | Subcommand | Key Capabilities |
| :--- | :--- | :--- |
| **Node Health Auditor** | `pve-tool node health` | CPU load avg, memory saturation, I/O wait latency pressure, ZFS/LVM storage pool capacity, kernel version, active guest counts. |
| **Cluster Auditor** | `pve-tool node cluster` | Quorum health verification, Corosync split-brain detection, offline node alarms, multi-node inventory. |
| **Backup Orchestrator** | `pve-tool backup run` | Automated `vzdump` triggers (`snapshot`/`suspend`/`stop`), multi-tier retention pruning (`keep-last`, `keep-daily`), dry-run simulation, JSON status logs. |
| **Live Snapshot Manager** | `pve-tool backup snapshot` | Instant non-disruptive live VM RAM/disk snapshots and LXC container freezing. |
| **Bulk Power Controller** | `pve-tool power execute` | Graceful shutdown countdowns, batch start/restart/stop filtered by tags (e.g. `k8s-node`, `prod`), resource pool ID, or node name. |
| **Security Baseline Auditor**| `pve-tool security audit` | Audits root password vs token auth, 2FA/TFA enforcement, cluster & guest firewall coverage, unauthenticated endpoint probes, SSL validity, and generates automated Bash remediation scripts. |

---

## 3. Quickstart & Installation

### Option A: Direct Python Virtualenv

```bash
# 1. Clone or download the repository
git clone https://github.com/algo2world/proxmox-pve-toolkit.git
cd proxmox-pve-toolkit

# 2. Initialize a Python virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install locked dependencies
pip install -r requirements.txt

# 4. Configure environment credentials
cp .env.example .env
# Edit .env with your Proxmox Host, User, and Token credentials
nano .env

# 5. Run configuration connectivity check
python main.py config-check
```

### Option B: Containerized via Docker / Docker Compose

```bash
# Build the minimal non-root image
docker build -t proxmox-pve-toolkit:latest .

# Run one-off node health audit
docker run --rm --env-file .env proxmox-pve-toolkit:latest node health

# Launch continuous hourly background audit daemon via Compose
docker compose up -d pve-auditor-scheduled
```

---

## 4. Proxmox API Token Setup Guide (Least-Privilege)

To follow security best practices, **do not use the root PAM password**. Create a dedicated system user and API Token with scoped ACL permissions.

### Proxmox Shell Commands:
```bash
# 1. Create a custom least-privilege role
pveum role add PVEToolkitRole -privs "VM.Audit VM.Backup VM.PowerMgmt VM.Config.Disk Sys.Audit Datacenter.Audit"

# 2. Create dedicated automation user in the PVE realm
pveum user add devops@pve --comment "DevOps Automation Service Account"

# 3. Grant the role across cluster root
pveum acl modify / -user devops@pve -role PVEToolkitRole

# 4. Generate API Token without privilege separation
pveum user token add devops@pve pve-toolkit-token --privsep 0
```

Copy the generated Secret Token UUID and populate your `.env`:
```env
PROXMOX_HOST=192.168.1.100
PROXMOX_PORT=8006
PROXMOX_USER=devops@pve
PROXMOX_TOKEN_NAME=pve-toolkit-token
PROXMOX_TOKEN_VALUE=00000000-0000-0000-0000-000000000000
VERIFY_SSL=false
```

---

## 5. Complete CLI Command Reference

### Global Helper Commands
```bash
# Display CLI version and metadata
python main.py version

# Validate API connection and list cluster nodes
python main.py config-check
```

### 1. Node & Cluster Health
```bash
# Full telemetry audit of default or specific node
python main.py node health --node pve-01

# Output raw JSON for Prometheus/Telegraf ingestion
python main.py node health --json

# Check whole cluster status, quorum, and node availability
python main.py node cluster
```

### 2. Automated Backups & Snapshots
```bash
# Dry-run backup for all guests on node (Safe preview)
python main.py backup run --dry-run

# Run snapshot backup for VMID 104 with retention pruning (keep 3 latest, 7 daily)
python main.py backup run --vmid 104 --storage local-zfs --mode snapshot --keep-last 3 --keep-daily 7

# Backup with status export to JSON log
python main.py backup run --vmid 200 --export-log /var/log/pve_backup_200.json

# Take immediate live snapshot with RAM state included
python main.py backup snapshot 101 pre-kernel-upgrade --desc "Snapshot before Linux 6.8 patch" --include-ram
```

### 3. Bulk Power State Management
```bash
# Graceful shutdown of all workloads tagged 'dev-stack'
python main.py power execute shutdown --tag dev-stack --timeout 45

# Graceful restart of all guests within resource pool 'k8s-cluster'
python main.py power execute reboot --pool k8s-cluster

# Start all stopped LXC containers on node pve-02
python main.py power execute start --node pve-02 --type lxc

# Dry-run simulation with automatic confirmation bypass
python main.py power execute stop --tag test-env --dry-run --yes
```

### 4. Security Baseline Auditor & Hardening
```bash
# Full security baseline audit (Firewall, Auth, 2FA, SSL, API Probes)
python main.py security audit

# Run audit and export automated remediation Bash fix script
python main.py security audit --export-script fix_hardening.sh

# View raw JSON compliance telemetry
python main.py security audit --json
```

---

## 6. Security Disclaimer & Best Practices

- **Self-Signed Certificates**: In private lab environments, set `VERIFY_SSL=false`. In production environments with Let's Encrypt or corporate root CA, set `VERIFY_SSL=true`.
- **Credential Storage**: Never commit `.env` containing secret API token values to public version control. Keep `.env` in `.gitignore`.
- **Privilege Separation**: Utilize token-based authentication (`devops@pve`) instead of root PAM credentials wherever possible.

---

## 7. About Algo2World & Ind. Ecosystem

**Proxmox PVE Toolkit** is engineered and maintained by **Nikil** and the **Algo2World** engineering team as part of the interconnected **Ind. Ecosystem** initiative.

### 🌐 The Ind. Ecosystem Suite

Explore our sovereign, privacy-centric open platforms and developer tools:

* **[algo2world.com](https://algo2world.com)** — Core algorithmic, distributed systems & AI engineering lab.
* **[samvad.chat](https://samvad.chat)** — Sovereign real-time communication & secure conversational matrix.
* **[ind.social](https://ind.social)** — Decentralized social federation & open discourse network.
* **[ind.network](https://ind.network)** — Next-generation distributed networking, mesh routing & edge telemetry.
* **[ind.center](https://ind.center)** — Unified identity, developer API gateways & knowledge registry.
* **[ind.trading](https://ind.trading)** — High-frequency trading infrastructure, quantitative engines & risk models.
* **[ind.report](https://ind.report)** — Investigative telemetry, data analytics & decentralized publishing.
* **[ind.shiksha](https://ind.shiksha)** — Open pedagogical universe, universal knowledge graph & adaptive learning systems.
* **[ind.quest](https://ind.quest)** — Interactive challenges, hackathons & skill discovery engine.
* **[ind.run](https://ind.run)** — Sovereign container orchestration, serverless execution & cloud fabric.
* **[ind.pet](https://ind.pet)** — Animal welfare registry, community shelter network & pet care directory.

---

## 8. Enterprise Support & Commercial Inquiries

Need custom infrastructure automation, bare-metal hypervisor hardening, or high-concurrency architecture consulting?

* **Engineering Lead / Founder:** Nikil (Algo2World)
* **Direct Email:** [nikil@algo2world.com](mailto:nikil@algo2world.com)
* **Telegram:** [@AUTO_GPT_BOT](https://t.me/AUTO_GPT_BOT)
* **Official Website:** [https://algo2world.com](https://algo2world.com)

**Services Available:**
- Enterprise Proxmox VE / Ceph / Kubernetes Cluster Architecture
- Custom Automated Disaster Recovery & Zero-RTO Replication Pipelines
- 24/7 Infrastructure SLA & Production Hardening Assessments

---

## 9. License:

Distributed under the **MIT License**. See `LICENSE` for details.  
Copyright (c) 2026 Nikil & Algo2World. All rights reserved.

