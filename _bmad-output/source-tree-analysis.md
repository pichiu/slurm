# Slurm Source Tree Analysis

> Generated: 2025-12-17 | Scan Level: Exhaustive

## Project Overview

- **Project Name**: Slurm Workload Manager
- **Version**: 26.05.0-0rc1
- **Repository Type**: Monolith
- **Primary Language**: C (C99)
- **Build System**: GNU Autotools

---

## Directory Structure

```
slurm/
├── src/                          # Main source code (679 .c files, 419 .h files)
│   ├── api/                      # Slurm C API library
│   ├── common/                   # Shared utilities and data structures
│   ├── conmgr/                   # Connection manager
│   ├── curl/                     # HTTP/CURL utilities
│   ├── database/                 # Database abstraction layer (MySQL)
│   ├── interfaces/               # Plugin interface definitions
│   ├── lua/                      # Lua scripting integration
│   │
│   ├── slurmctld/                # Central control daemon [DAEMON]
│   ├── slurmd/                   # Node daemon [DAEMON]
│   │   ├── slurmd/               # Main slurmd code
│   │   ├── slurmstepd/           # Job step manager
│   │   └── common/               # Shared slurmd components
│   ├── slurmdbd/                 # Database daemon [DAEMON]
│   ├── slurmrestd/               # REST API daemon [DAEMON]
│   │   └── plugins/              # REST API plugins
│   │       ├── auth/             # JWT, local auth
│   │       └── openapi/          # OpenAPI endpoint handlers
│   ├── sackd/                    # Accounting keepalive daemon [DAEMON]
│   │
│   ├── sbatch/                   # Batch job submission [CLI]
│   ├── srun/                     # Parallel job launcher [CLI]
│   ├── salloc/                   # Interactive allocation [CLI]
│   ├── sattach/                  # Job attachment [CLI]
│   ├── scancel/                  # Job cancellation [CLI]
│   ├── sbcast/                   # File broadcast [CLI]
│   ├── squeue/                   # Job queue display [CLI]
│   ├── sinfo/                    # Cluster info display [CLI]
│   ├── sstat/                    # Running job stats [CLI]
│   ├── sacct/                    # Accounting reports [CLI]
│   ├── sdiag/                    # Diagnostics [CLI]
│   ├── sprio/                    # Priority display [CLI]
│   ├── scontrol/                 # Admin tool [CLI]
│   ├── sacctmgr/                 # Accounting management [CLI]
│   ├── sreport/                  # Accounting reports [CLI]
│   ├── sshare/                   # Fair-share info [CLI]
│   ├── strigger/                 # Event triggers [CLI]
│   ├── scrontab/                 # Cron scheduling [CLI]
│   ├── scrun/                    # Container launcher [CLI]
│   ├── stepmgr/                  # Step management [CLI]
│   ├── sview/                    # GUI monitoring [CLI]
│   ├── bcast/                    # Broadcast library
│   │
│   └── plugins/                  # Plugin system (38 categories)
│       ├── accounting_storage/   # Database backends
│       ├── acct_gather_*/        # Accounting collectors
│       ├── auth/                 # Authentication (munge, jwt, none, slurm)
│       ├── burst_buffer/         # Burst buffer management
│       ├── cgroup/               # Cgroup v1/v2 support
│       ├── cli_filter/           # CLI command filtering
│       ├── cred/                 # Credential management
│       ├── data_parser/          # Data parsing plugins
│       ├── gpu/                  # GPU management (nvidia, amd, intel)
│       ├── gres/                 # Generic resources
│       ├── hash/                 # Hash algorithms
│       ├── job_submit/           # Job submission hooks
│       ├── jobacct_gather/       # Job accounting
│       ├── jobcomp/              # Job completion logging
│       ├── mcs/                  # Multi-Category Security
│       ├── mpi/                  # MPI support (pmix, pmi2)
│       ├── node_features/        # Node feature management
│       ├── preempt/              # Job preemption
│       ├── priority/             # Priority calculation
│       ├── proctrack/            # Process tracking
│       ├── sched/                # Schedulers (builtin, backfill)
│       ├── select/               # Resource selection (linear, cons_tres)
│       ├── serializer/           # Data serialization
│       ├── switch/               # Network switch support
│       ├── task/                 # Task management
│       ├── tls/                  # TLS encryption
│       └── topology/             # Cluster topology
│
├── slurm/                        # Public header files
│   ├── slurm.h                   # Main API header
│   ├── slurmdb.h                 # Accounting database API
│   ├── slurm_errno.h             # Error codes
│   ├── spank.h                   # SPANK plugin API
│   └── pmi.h                     # PMI interface
│
├── doc/                          # Documentation
│   ├── html/                     # 91 HTML documentation pages
│   └── man/                      # Man pages
│       ├── man1/                 # 22 command manuals
│       ├── man5/                 # 15 config file manuals
│       └── man8/                 # 7 admin manuals
│
├── etc/                          # Configuration examples
│   └── slurm.conf.example        # Sample configuration
│
├── testsuite/                    # Test suite
│   ├── expect/                   # Expect-based functional tests
│   ├── python/                   # Python tests
│   └── slurm_unit/               # Unit tests (Check framework)
│
├── contribs/                     # Contributed modules
│   ├── lua/                      # Lua bindings
│   ├── perlapi/                  # Perl API
│   ├── pam/                      # PAM modules
│   ├── pam_slurm_adopt/          # PAM adoption module
│   ├── pmi/                      # PMI library
│   ├── pmi2/                     # PMI2 library
│   ├── torque/                   # Torque compatibility
│   └── openlava/                 # OpenLava compatibility
│
├── auxdir/                       # Autotools support files
│   ├── *.m4                      # Autoconf macros
│   └── slurm.m4                  # Slurm-specific macros
│
├── debian/                       # Debian packaging
│
├── tools/                        # Build and utility scripts
│
├── configure.ac                  # Autoconf configuration
├── Makefile.am                   # Automake template
├── slurm.spec                    # RPM spec file
├── META                          # Version metadata
├── README.md                     # Project readme
├── INSTALL                       # Installation instructions
├── CONTRIBUTING.md               # Contribution guidelines
├── COPYING                       # GPL license
└── CHANGELOG/                    # Version changelogs
```

---

## Critical Directories

### 1. Core Daemons (`src/slurmctld/`, `src/slurmd/`, `src/slurmdbd/`, `src/slurmrestd/`)

| Directory | Entry Point | Purpose |
|-----------|-------------|---------|
| `src/slurmctld/` | `controller.c:main()` | Central controller - job scheduling, resource management |
| `src/slurmd/slurmd/` | `slurmd.c:main()` | Node daemon - local job execution |
| `src/slurmd/slurmstepd/` | `slurmstepd.c:main()` | Job step manager - task execution |
| `src/slurmdbd/` | `slurmdbd.c:main()` | Database daemon - accounting |
| `src/slurmrestd/` | `slurmrestd.c:main()` | REST API daemon - HTTP interface |
| `src/sackd/` | `sackd.c:main()` | Accounting keepalive daemon |

### 2. CLI Tools (`src/s*/`)

| Tool | File | Purpose |
|------|------|---------|
| sbatch | `src/sbatch/sbatch.c` | Submit batch jobs |
| srun | `src/srun/srun.c` | Run parallel jobs |
| salloc | `src/salloc/salloc.c` | Interactive allocation |
| squeue | `src/squeue/squeue.c` | View job queue |
| sinfo | `src/sinfo/sinfo.c` | View cluster info |
| scontrol | `src/scontrol/scontrol.c` | Admin control |
| sacct | `src/sacct/sacct.c` | Accounting reports |
| scancel | `src/scancel/scancel.c` | Cancel jobs |

### 3. Common Libraries (`src/common/`)

Critical files:
- `job_record.h/c` - Job data structures
- `node_conf.h/c` - Node configuration
- `part_record.h/c` - Partition management
- `slurm_protocol_*.h/c` - RPC protocol
- `pack.h/c` - Serialization
- `log.h/c` - Logging
- `xmalloc.h/c` - Memory management

### 4. Plugin System (`src/plugins/`)

38 plugin categories with 100+ implementations. Key categories:

| Category | Implementations | Purpose |
|----------|-----------------|---------|
| `auth/` | jwt, munge, none, slurm | Authentication |
| `select/` | linear, cons_tres | Resource selection |
| `sched/` | builtin, backfill | Job scheduling |
| `accounting_storage/` | mysql, slurmdbd, ctld_relay | Accounting storage |
| `cgroup/` | v1, v2 | Resource isolation |

---

## Key File Locations

### Configuration Files
- `etc/slurm.conf.example` - Main configuration template
- `etc/slurmdbd.conf.example` - Database daemon config
- `etc/cgroup.conf.example` - Cgroup configuration

### Public Headers
- `slurm/slurm.h` - Main Slurm API
- `slurm/slurmdb.h` - Accounting API
- `slurm/spank.h` - SPANK plugin API

### Entry Points
- `src/slurmctld/controller.c` - Controller daemon
- `src/slurmd/slurmd/slurmd.c` - Node daemon
- `src/slurmrestd/slurmrestd.c` - REST daemon

---

## Statistics

| Metric | Count |
|--------|-------|
| C source files (.c) | 679 |
| Header files (.h) | 419 |
| Plugin categories | 38 |
| CLI tools | 19 |
| Daemons | 6 |
| HTML doc pages | 91 |
| Man pages | 44 |
| Lines of code (est.) | 500,000+ |
