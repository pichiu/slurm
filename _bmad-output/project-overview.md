# Slurm Workload Manager - Project Overview

> Generated: 2025-12-17 | Version: 26.05.0-0rc1

## What is Slurm?

Slurm (Simple Linux Utility for Resource Management) is an open-source, highly scalable cluster management and job scheduling system for Linux clusters. Originally developed at Lawrence Livermore National Laboratory, it is now maintained by SchedMD LLC.

**Key Capabilities:**
- Allocates exclusive/non-exclusive access to compute nodes
- Provides framework for starting, executing, and monitoring parallel jobs
- Arbitrates conflicting resource requests via a job queue
- Supports clusters from small workgroups to millions of cores

---

## Quick Reference

| Property | Value |
|----------|-------|
| **Project Name** | Slurm Workload Manager |
| **Version** | 26.05.0-0rc1 |
| **Protocol Version** | v45 |
| **Primary Language** | C (C99) |
| **Build System** | GNU Autotools |
| **License** | GPLv2+ |
| **Repository Type** | Monolith |
| **Project Types** | backend, infra, cli |

### Technology Stack

| Category | Technology |
|----------|------------|
| Core Language | C (C99) |
| Database | MySQL / MariaDB 5.0+ |
| Authentication | MUNGE (default), JWT |
| Build | autoconf, automake, libtool |
| Testing | Check, Expect, Pytest |
| Serialization | Custom binary, JSON, YAML |

### Network Ports

| Service | Port |
|---------|------|
| slurmctld | 6817 |
| slurmd | 6818 |
| slurmdbd | 6819 |
| slurmrestd | 6820 |

---

## Architecture Summary

```
Users → CLI Tools/REST API → slurmctld (Controller)
                                    ↓
                             slurmd (Nodes)
                                    ↓
                          slurmstepd (Tasks)
                                    ↓
                            slurmdbd → MySQL
```

### Core Components

| Component | Purpose |
|-----------|---------|
| **slurmctld** | Central controller - scheduling, allocation |
| **slurmd** | Node daemon - job execution |
| **slurmstepd** | Step manager - task management |
| **slurmdbd** | Database daemon - accounting |
| **slurmrestd** | REST API daemon - HTTP interface |

### CLI Tools

| Category | Tools |
|----------|-------|
| Job Submission | sbatch, srun, salloc |
| Job Control | scancel, sattach, sbcast |
| Monitoring | squeue, sinfo, sstat, sacct, sdiag |
| Administration | scontrol, sacctmgr, sreport |
| Utilities | sprio, sshare, strigger, scrontab |

---

## Project Statistics

| Metric | Count |
|--------|-------|
| C Source Files | 679 |
| Header Files | 419 |
| Plugin Categories | 38 |
| CLI Tools | 19 |
| Daemons | 6 |
| HTML Doc Pages | 91 |
| Man Pages | 44 |
| Estimated LOC | 500,000+ |

---

## Key Features

### Scheduling
- Multiple scheduling plugins (FIFO, backfill)
- Fair-share scheduling
- Job priority with multiple factors
- Preemption support
- Reservations and maintenance windows

### Resource Management
- CPU, memory, GPU allocation
- Generic resource (GRES) framework
- Consumable resources tracking
- Topology-aware scheduling
- NUMA-aware placement

### Accounting
- Job completion logging
- Resource usage tracking
- User/account/QoS hierarchies
- Fair-share calculations
- Multi-cluster federation

### Security
- MUNGE authentication
- JWT token support
- Job credential signing
- Resource limit enforcement
- Cgroup isolation

---

## Getting Started

### For Users

```bash
# Submit a batch job
sbatch myjob.sh

# Run an interactive job
srun --pty bash

# Check job queue
squeue -u $USER

# Check cluster status
sinfo
```

### For Administrators

```bash
# Configure: Edit /etc/slurm/slurm.conf
# Start services:
systemctl start slurmctld
systemctl start slurmd

# Check diagnostics
sdiag
scontrol show config
```

### For Developers

```bash
# Build from source
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
```

---

## Documentation Links

- **Official Site**: https://slurm.schedmd.com/
- **Admin Guide**: https://slurm.schedmd.com/quickstart_admin.html
- **User Guide**: https://slurm.schedmd.com/quickstart.html
- **API Reference**: https://slurm.schedmd.com/api.html
- **Issue Tracker**: https://support.schedmd.com/

---

## Related Files in This Documentation Set

- [Architecture](./architecture.md) - System architecture details
- [Source Tree Analysis](./source-tree-analysis.md) - Code organization
- [API Contracts](./api-contracts.md) - REST API endpoints
- [Data Models](./data-models.md) - Database structures
- [Development Guide](./development-guide.md) - Build and contribute
