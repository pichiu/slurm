# Slurm Architecture Documentation

> Generated: 2025-12-17 | Scan Level: Exhaustive

## Executive Summary

Slurm (Simple Linux Utility for Resource Management) is a highly scalable, fault-tolerant cluster management and job scheduling system for Linux clusters. It is designed for HPC (High-Performance Computing) environments and can manage clusters ranging from small workgroups to massive supercomputers with millions of cores.

**Key Characteristics:**
- Open-source (GPLv2+)
- Written in C (C99)
- Plugin-based architecture
- Distributed master-worker design
- Protocol version: v45 (26.05)

---

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Interface Layer                          │
│  ┌─────┐ ┌─────┐ ┌──────┐ ┌──────┐ ┌───────┐ ┌──────┐ ┌─────────┐  │
│  │sbatch│ │srun │ │salloc│ │squeue│ │scontrol│ │sacct │ │slurmrestd│ │
│  └──┬──┘ └──┬──┘ └──┬───┘ └──┬───┘ └───┬───┘ └──┬───┘ └────┬────┘  │
└─────┼───────┼───────┼────────┼─────────┼────────┼──────────┼────────┘
      │       │       │        │         │        │          │
      └───────┴───────┴────────┴────┬────┴────────┴──────────┘
                                    │ RPC Protocol (port 6817)
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Control Plane (slurmctld)                       │
│  ┌───────────┐ ┌────────────┐ ┌──────────┐ ┌────────────────────┐   │
│  │ Job Mgr   │ │ Node Mgr   │ │ Scheduler │ │ Partition Mgr      │   │
│  └───────────┘ └────────────┘ └──────────┘ └────────────────────┘   │
│  ┌───────────┐ ┌────────────┐ ┌──────────┐ ┌────────────────────┐   │
│  │ State Save│ │ Backup Mgr │ │ RPC Mgr  │ │ Plugin Manager     │   │
│  └───────────┘ └────────────┘ └──────────┘ └────────────────────┘   │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ RPC Protocol (port 6818)
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Compute Plane (slurmd)                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Node 1        │ Node 2        │ Node 3        │ Node N      │    │
│  │ ┌───────────┐ │ ┌───────────┐ │ ┌───────────┐ │ ┌─────────┐ │    │
│  │ │  slurmd   │ │ │  slurmd   │ │ │  slurmd   │ │ │ slurmd  │ │    │
│  │ │    │      │ │ │    │      │ │ │    │      │ │ │    │    │ │    │
│  │ │slurmstepd │ │ │slurmstepd │ │ │slurmstepd │ │ │stepd(s) │ │    │
│  │ └───────────┘ │ └───────────┘ │ └───────────┘ │ └─────────┘ │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ SQL Protocol (port 6819)
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Accounting Plane (slurmdbd)                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                  slurmdbd (Database Daemon)                     │ │
│  │  ┌────────────┐ ┌───────────────┐ ┌──────────────────────────┐ │ │
│  │  │ RPC Handler│ │ Job Accounting│ │ Association/QoS Manager  │ │ │
│  │  └────────────┘ └───────────────┘ └──────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    MySQL / MariaDB                              │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Daemons

### 1. slurmctld (Central Controller)

**Purpose**: Brain of the cluster - manages jobs, nodes, partitions, and scheduling

**Entry Point**: `src/slurmctld/controller.c`

**Key Responsibilities:**
- Job queue management
- Resource allocation decisions
- Node state tracking
- Partition management
- Scheduling algorithm execution
- State persistence and recovery
- High availability (backup controllers)

**Key Components:**
| Component | File | Purpose |
|-----------|------|---------|
| Job Manager | `job_mgr.c` | Job lifecycle management |
| Job Scheduler | `job_scheduler.c` | Scheduling decisions |
| Node Manager | `node_mgr.c` | Node state tracking |
| Partition Manager | `partition_mgr.c` | Partition configuration |
| State Save | `state_save.c` | Checkpoint/restart |
| Backup | `backup.c` | HA failover |

**Default Port**: 6817

### 2. slurmd (Node Daemon)

**Purpose**: Agent on each compute node - executes jobs locally

**Entry Point**: `src/slurmd/slurmd/slurmd.c`

**Key Responsibilities:**
- Accept job allocations from slurmctld
- Launch job steps via slurmstepd
- Monitor local resource usage
- Report node health status
- Enforce resource limits

**Key Components:**
| Component | File | Purpose |
|-----------|------|---------|
| Request Handler | `req.c` | RPC processing |
| Machine Stats | `get_mach_stat.c` | Hardware detection |
| Job Memory | `job_mem_limit.c` | Memory enforcement |
| Credential | `cred_context.c` | Job authentication |

**Default Port**: 6818

### 3. slurmstepd (Step Manager)

**Purpose**: Manages individual job step execution

**Entry Point**: `src/slurmd/slurmstepd/slurmstepd.c`

**Key Responsibilities:**
- Task launching and management
- I/O redirection (stdin/stdout/stderr)
- Signal handling
- Process tracking
- Resource accounting
- Container/namespace support

**Key Components:**
| Component | File | Purpose |
|-----------|------|---------|
| Task Manager | `task.c` | Process launch |
| I/O Handler | `io.c` | Stream management |
| Container | `container.c` | Namespace setup |
| X11 Forward | `x11_forwarding.c` | Display forwarding |

### 4. slurmdbd (Database Daemon)

**Purpose**: Central accounting database service

**Entry Point**: `src/slurmdbd/slurmdbd.c`

**Key Responsibilities:**
- Job completion logging
- Usage accounting
- Association/QoS management
- Fair-share calculation data
- Multi-cluster federation

**Default Port**: 6819

### 5. slurmrestd (REST API Daemon)

**Purpose**: HTTP/REST interface to Slurm

**Entry Point**: `src/slurmrestd/slurmrestd.c`

**Key Responsibilities:**
- OpenAPI specification compliance
- JWT authentication
- JSON/YAML serialization
- Proxy to slurmctld/slurmdbd

**Default Port**: 6820

---

## Plugin Architecture

Slurm uses a highly modular plugin system allowing customization of nearly every aspect.

### Plugin Categories (38 total)

```
plugins/
├── Authentication & Security
│   ├── auth/           # Authentication (munge, jwt, none, slurm)
│   ├── cred/           # Job credentials
│   ├── tls/            # TLS encryption
│   ├── certgen/        # Certificate generation
│   └── certmgr/        # Certificate management
│
├── Scheduling & Selection
│   ├── sched/          # Schedulers (builtin, backfill)
│   ├── select/         # Resource selection (linear, cons_tres)
│   ├── priority/       # Priority calculation
│   └── preempt/        # Job preemption
│
├── Resource Management
│   ├── gres/           # Generic resources
│   ├── gpu/            # GPU management (nvidia, amd, intel)
│   ├── burst_buffer/   # Burst buffer
│   ├── node_features/  # Node features
│   └── topology/       # Network topology
│
├── Accounting & Storage
│   ├── accounting_storage/  # Database backends
│   ├── jobacct_gather/      # Job accounting
│   ├── jobcomp/             # Job completion
│   └── acct_gather_*/       # Energy, filesystem, network
│
├── Job Execution
│   ├── task/           # Task affinity, cgroups
│   ├── proctrack/      # Process tracking
│   ├── mpi/            # MPI support (pmix, pmi2)
│   ├── cgroup/         # Cgroup v1/v2
│   └── namespace/      # Linux namespaces
│
└── Data & Utilities
    ├── serializer/     # Data serialization
    ├── data_parser/    # API data parsing
    ├── hash/           # Hash algorithms
    ├── job_submit/     # Submission hooks
    └── cli_filter/     # CLI filtering
```

### Plugin Interface

All plugins implement standard symbols:
```c
const char plugin_name[];          // Human-readable name
const char plugin_type[];          // Plugin type identifier
const uint32_t plugin_version;     // Slurm version compatibility
```

---

## Communication Protocol

### RPC Protocol

Slurm uses a custom binary RPC protocol:

- **Serialization**: Custom pack/unpack (`src/common/pack.c`)
- **Protocol Version**: Negotiated per connection
- **Encryption**: Optional TLS
- **Authentication**: MUNGE or JWT tokens

### Message Flow

```
Client → pack(request) → encrypt → send → slurmctld
slurmctld → decrypt → unpack → process → pack(response) → encrypt → send → Client
```

### Default Ports

| Service | Port | Protocol |
|---------|------|----------|
| slurmctld | 6817 | Slurm RPC |
| slurmd | 6818 | Slurm RPC |
| slurmdbd | 6819 | Slurm RPC |
| slurmrestd | 6820 | HTTP/HTTPS |

---

## Data Flow

### Job Submission Flow

```
1. User submits job (sbatch/srun/salloc)
2. CLI validates and sends to slurmctld
3. slurmctld:
   a. Authenticates request
   b. Validates against associations/QoS
   c. Queues job
   d. Scheduler evaluates priority
4. When resources available:
   a. slurmctld allocates nodes
   b. Sends allocation to slurmd(s)
5. slurmd spawns slurmstepd
6. slurmstepd launches user tasks
7. On completion:
   a. slurmd reports to slurmctld
   b. slurmctld records in slurmdbd
```

### Accounting Flow

```
Job Event → slurmctld → slurmdbd → MySQL
                ↓
         State files (backup)
```

---

## High Availability

### Controller Failover

- Primary and backup slurmctld daemons
- State saved periodically to shared storage
- Automatic failover on primary failure
- Configurable via `BackupController` in slurm.conf

### Database Redundancy

- slurmdbd can use MySQL replication
- Optional in-memory caching
- Graceful handling of database outages

---

## Scalability

Slurm is designed for extreme scale:

| Metric | Capability |
|--------|------------|
| Nodes | 100,000+ |
| Cores | Millions |
| Jobs/day | Millions |
| Concurrent jobs | 100,000+ |

### Optimization Techniques

- Hierarchical communication
- Aggregated node updates
- Efficient bit-string operations
- Message coalescing
- Tree-based fan-out

---

## Security Model

### Authentication

1. **MUNGE** (default) - Shared-key authentication
2. **JWT** - Token-based authentication
3. **Slurm internal** - Native authentication

### Authorization

- User/account/QoS hierarchy
- Partition access controls
- Resource limits
- Job credential signing

### Network Security

- Optional TLS encryption
- Configurable allowed networks
- Firewall-friendly port configuration

---

## Configuration Files

| File | Purpose |
|------|---------|
| `slurm.conf` | Main configuration |
| `slurmdbd.conf` | Database daemon config |
| `cgroup.conf` | Cgroup settings |
| `gres.conf` | Generic resources |
| `topology.conf` | Network topology |

---

## Technology Stack Summary

| Category | Technology |
|----------|------------|
| Language | C (C99) |
| Build | GNU Autotools |
| Database | MySQL/MariaDB |
| Auth | MUNGE, JWT |
| Serialization | Custom binary, JSON, YAML |
| Process Control | cgroups v1/v2, procfs |
| Networking | TCP/IP, Unix sockets |
| GPU Support | NVIDIA NVML, AMD ROCm, Intel OneAPI |
| MPI | PMIx, PMI2 |
| Containers | Singularity, Docker (via plugins) |
