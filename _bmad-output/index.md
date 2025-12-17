# Slurm Workload Manager - Documentation Index

> Generated: 2025-12-17 | Scan Level: Exhaustive | Version: 26.05.0-0rc1

This documentation set was generated for AI-assisted development of the Slurm Workload Manager codebase. It provides comprehensive reference materials optimized for LLM context efficiency.

---

## Quick Start

| I want to... | Go to |
|--------------|-------|
| Understand what Slurm is | [Project Overview](./project-overview.md) |
| Learn the system architecture | [Architecture](./architecture.md) |
| Build from source | [Development Guide](./development-guide.md) |
| Navigate the source code | [Source Tree Analysis](./source-tree-analysis.md) |
| Work with the REST API | [API Contracts](./api-contracts.md) |
| Understand data structures | [Data Models](./data-models.md) |

---

## Generated Documentation

### Core Documents

| Document | Description | Status |
|----------|-------------|--------|
| [Project Overview](./project-overview.md) | Quick reference, capabilities, getting started | Complete |
| [Architecture](./architecture.md) | System design, daemons, plugins, communication | Complete |
| [Source Tree Analysis](./source-tree-analysis.md) | Directory structure, key files, statistics | Complete |
| [Development Guide](./development-guide.md) | Build instructions, coding style, testing | Complete |
| [API Contracts](./api-contracts.md) | REST API endpoints for slurmctld and slurmdbd | Complete |
| [Data Models](./data-models.md) | Core data structures and database schema | Complete |

### State Files

| File | Purpose |
|------|---------|
| [project-scan-report.json](./project-scan-report.json) | Workflow state and findings |

---

## Project Existing Documentation

### Official Documentation (in repository)

| Location | Content |
|----------|---------|
| `doc/html/` | 91 HTML documentation pages |
| `doc/man/` | 44 man pages |
| `README.md` | Project introduction |
| `INSTALL` | Installation instructions |
| `CONTRIBUTING.md` | Contribution guidelines |
| `CHANGELOG.md` | Version history |
| `SECURITY.md` | Security policy |

### External Resources

- **Official Site**: https://slurm.schedmd.com/
- **Admin Guide**: https://slurm.schedmd.com/quickstart_admin.html
- **User Guide**: https://slurm.schedmd.com/quickstart.html
- **API Reference**: https://slurm.schedmd.com/api.html
- **Issue Tracker**: https://support.schedmd.com/

---

## Project Summary

### Classification

| Property | Value |
|----------|-------|
| Repository Type | Monolith |
| Primary Type | Backend (infrastructure + CLI) |
| Architecture | Distributed master-worker |
| Primary Language | C (C99) |
| Build System | GNU Autotools |

### Key Statistics

| Metric | Count |
|--------|-------|
| C Source Files | 679 |
| Header Files | 419 |
| Plugin Categories | 38 |
| CLI Tools | 19 |
| Daemons | 6 |
| Estimated LOC | 500,000+ |

### Technology Stack

| Category | Technology |
|----------|------------|
| Language | C (C99) |
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

## Architecture Overview

```
Users --> CLI Tools/REST API --> slurmctld (Controller)
                                       |
                                 slurmd (Nodes)
                                       |
                               slurmstepd (Tasks)
                                       |
                                 slurmdbd --> MySQL
```

### Core Components

| Component | Purpose | Entry Point |
|-----------|---------|-------------|
| slurmctld | Central controller - scheduling, allocation | `src/slurmctld/controller.c` |
| slurmd | Node daemon - job execution | `src/slurmd/slurmd/slurmd.c` |
| slurmstepd | Step manager - task management | `src/slurmd/slurmstepd/slurmstepd.c` |
| slurmdbd | Database daemon - accounting | `src/slurmdbd/slurmdbd.c` |
| slurmrestd | REST API daemon - HTTP interface | `src/slurmrestd/slurmrestd.c` |

### CLI Tools

| Category | Tools |
|----------|-------|
| Job Submission | sbatch, srun, salloc |
| Job Control | scancel, sattach, sbcast |
| Monitoring | squeue, sinfo, sstat, sacct, sdiag |
| Administration | scontrol, sacctmgr, sreport |
| Utilities | sprio, sshare, strigger, scrontab |

---

## Source Navigation

### Critical Directories

| Directory | Purpose |
|-----------|---------|
| `src/slurmctld/` | Controller daemon |
| `src/slurmd/` | Node daemon and step daemon |
| `src/slurmdbd/` | Database daemon |
| `src/slurmrestd/` | REST API daemon |
| `src/common/` | Shared libraries |
| `src/interfaces/` | Plugin interfaces |
| `src/plugins/` | 38 plugin categories |
| `src/api/` | Client API |
| `slurm/` | Public headers |

### Key Files for Understanding

| File | Purpose |
|------|---------|
| `src/common/job_record.h` | Job data structure |
| `src/common/node_conf.h` | Node data structure |
| `src/common/part_record.h` | Partition data structure |
| `slurm/slurm.h` | Main public API |
| `slurm/slurmdb.h` | Accounting API |
| `src/common/slurm_protocol_defs.h` | RPC definitions |
| `src/common/pack.c` | Serialization |

---

## Development Quick Reference

### Build

```bash
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
```

### Debug Build

```bash
./configure --prefix=/usr/local \
            --enable-debug \
            --enable-developer \
            CFLAGS="-g -O0"
make -j$(nproc)
```

### Testing

```bash
# Unit tests
cd testsuite/slurm_unit && make check

# Functional tests (requires running cluster)
cd testsuite/expect && ./regression.py

# Python tests
cd testsuite/python && pytest
```

---

## Document Revision History

| Date | Action | Notes |
|------|--------|-------|
| 2025-12-17 | Initial generation | Exhaustive scan completed |

---

## AI Development Notes

This documentation is optimized for AI-assisted development:

1. **Project Overview** - Start here for context
2. **Architecture** - Understand system design before modifications
3. **Source Tree Analysis** - Find relevant files quickly
4. **Data Models** - Understand core structures before coding
5. **API Contracts** - Reference for REST API work
6. **Development Guide** - Follow coding standards

### Common Tasks

| Task | Recommended Reading |
|------|---------------------|
| Add new CLI option | Development Guide > Common Development Tasks |
| Add new RPC | Architecture > Communication Protocol, Development Guide |
| Create new plugin | Architecture > Plugin Architecture, Development Guide |
| Modify job handling | Data Models > job_record_t, Source Tree > slurmctld |
| Work with REST API | API Contracts, Source Tree > slurmrestd |
