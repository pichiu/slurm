# Slurm Data Models

> Generated: 2025-12-17 | Source: Exhaustive code analysis

## Overview

Slurm uses a layered data model architecture:
1. **Runtime Layer** (slurmctld) - Active job, node, partition management
2. **Accounting Layer** (slurmdbd) - Historical records, usage tracking
3. **Persistence Layer** (MySQL) - Long-term data storage

---

## Core Runtime Data Structures

### 1. job_record_t (Job Record)

**Location**: `src/common/job_record.h`

The primary data structure for representing jobs in slurmctld.

```c
// Key fields (simplified)
struct job_record {
    // Identification
    uint32_t job_id;
    uint32_t array_job_id;
    uint32_t array_task_id;
    uint32_t het_job_id;

    // User/Account
    char *account;
    uint32_t user_id;
    char *user_name;
    uint32_t group_id;

    // Resources
    job_resources_t *job_resrcs;
    bitstr_t *node_bitmap;
    char *nodes;
    uint32_t node_cnt;
    uint32_t total_cpus;
    uint64_t pn_min_memory;

    // Time
    time_t submit_time;
    time_t start_time;
    time_t end_time;
    time_t suspend_time;
    uint32_t time_limit;

    // State
    uint32_t job_state;
    uint16_t state_reason;
    uint32_t priority;
    uint32_t exit_code;

    // References
    job_details_t *details;
    List step_list;
    part_record_t *part_ptr;
    slurmdb_assoc_rec_t *assoc_ptr;
    slurmdb_qos_rec_t *qos_ptr;
};
```

### 2. node_record_t (Node Record)

**Location**: `src/common/node_conf.h`

Represents a compute node in the cluster.

```c
struct node_record {
    // Identification
    char *name;
    char *comm_name;
    char *node_hostname;
    uint32_t index;

    // Hardware
    uint16_t cpus;
    uint16_t cpus_efctv;
    uint16_t cores;
    uint16_t threads;
    uint16_t tot_sockets;
    uint64_t real_memory;
    uint32_t tmp_disk;

    // State
    uint32_t node_state;
    uint32_t next_state;
    char *reason;
    time_t reason_time;
    time_t last_response;

    // Load
    float cpu_load;
    uint64_t free_mem;
    uint32_t up_time;

    // Resources
    List gres_list;
    uint64_t *tres_cnt;
    char *tres_str;

    // Partition associations
    part_record_t **part_pptr;
    config_record_t *config_ptr;
};
```

### 3. part_record_t (Partition Record)

**Location**: `src/common/part_record.h`

Represents a partition (queue) in the cluster.

```c
struct part_record {
    // Identification
    char *name;
    char *nodes;
    bitstr_t *node_bitmap;

    // Capacity
    uint32_t total_nodes;
    uint32_t total_cpus;
    uint32_t max_nodes;
    uint32_t min_nodes;

    // Limits
    uint32_t max_time;
    uint32_t default_time;
    uint32_t max_cpus_per_node;
    uint64_t max_mem_per_cpu;
    uint16_t max_share;

    // Priority
    uint16_t priority_job_factor;
    uint16_t priority_tier;

    // State
    uint16_t state_up;
    uint32_t flags;

    // QoS
    slurmdb_qos_rec_t *qos_ptr;
    char *billing_weights;
};
```

### 4. job_resources_t (Job Resource Allocation)

**Location**: `src/common/job_resources.h`

Tracks detailed resource allocation for a job.

```c
struct job_resources {
    // CPU allocation
    uint16_t *cpus;
    uint32_t cpu_array_cnt;
    uint16_t *cpu_array_value;
    uint32_t *cpu_array_reps;

    // Core information
    bitstr_t *core_bitmap;
    uint16_t *cores_per_socket;
    uint16_t *sockets_per_node;

    // Memory
    uint64_t *memory_allocated;
    uint64_t *memory_used;

    // Node allocation
    bitstr_t *node_bitmap;
    char *nodes;
    uint32_t nhosts;
    uint32_t ncpus;

    // Usage tracking
    uint16_t *cpus_used;
    bitstr_t *core_bitmap_used;
};
```

---

## Accounting Data Structures (slurmdb)

**Location**: `slurm/slurmdb.h`

### 1. slurmdb_assoc_rec_t (Association Record)

Links users, accounts, clusters, and partitions with resource limits.

```c
struct slurmdb_assoc_rec {
    // Identification
    uint32_t id;
    char *acct;
    char *user;
    char *cluster;
    char *partition;
    uid_t uid;

    // Hierarchy
    char *parent_acct;
    uint32_t parent_id;
    char *lineage;

    // Group limits
    uint32_t grp_jobs;
    uint32_t grp_submit_jobs;
    char *grp_tres;
    char *grp_tres_mins;
    char *grp_tres_run_mins;
    uint32_t grp_wall;

    // Individual limits
    uint32_t max_jobs;
    uint32_t max_submit_jobs;
    char *max_tres_pj;
    char *max_tres_run_mins;
    uint32_t max_wall_pj;

    // QoS
    uint32_t def_qos_id;
    List qos_list;

    // Usage
    slurmdb_assoc_usage_t *usage;
    uint32_t shares_raw;
};
```

### 2. slurmdb_job_rec_t (Database Job Record)

Historical job record stored in accounting database.

```c
struct slurmdb_job_rec {
    // Identification
    uint32_t jobid;
    uint32_t array_job_id;
    uint32_t array_task_id;
    char *cluster;
    uint32_t het_job_id;

    // Account info
    char *account;
    uid_t uid;
    char *user;
    uint32_t associd;

    // Timestamps
    time_t submit;
    time_t start;
    time_t end;
    time_t eligible;

    // Resources
    uint32_t req_cpus;
    uint64_t req_mem;
    char *tres_alloc_str;
    char *tres_req_str;

    // State
    uint32_t state;
    int32_t exitcode;
    int32_t derived_ec;

    // CPU time
    uint32_t user_cpu_sec;
    uint32_t sys_cpu_sec;
    uint32_t tot_cpu_sec;

    // Allocation
    char *alloc_nodes;
    char *nodes;
    char *partition;

    // Steps
    List steps;
};
```

### 3. slurmdb_qos_rec_t (QoS Record)

Quality of Service configuration.

```c
struct slurmdb_qos_rec {
    // Identification
    uint32_t id;
    char *name;
    char *description;

    // Priority
    uint32_t priority;
    double norm_priority;
    uint16_t preempt_mode;
    double usage_factor;

    // Group limits
    uint32_t grp_jobs;
    uint32_t grp_submit_jobs;
    char *grp_tres;
    char *grp_tres_mins;
    char *grp_tres_run_mins;
    uint32_t grp_wall;

    // Per-account/user limits
    uint32_t max_jobs_pa;
    uint32_t max_jobs_pu;
    uint32_t max_submit_jobs_pa;
    uint32_t max_submit_jobs_pu;

    // Per-job limits
    char *max_tres_pj;
    char *max_tres_mins_pj;
    uint32_t max_wall_pj;

    // Preemption
    List preempt_list;
    bitstr_t *preempt_bitstr;

    // Usage
    slurmdb_qos_usage_t *usage;
};
```

### 4. slurmdb_tres_rec_t (TRES Record)

Trackable Resource types.

```c
struct slurmdb_tres_rec {
    uint32_t id;
    char *type;    // cpu, mem, energy, node, billing, etc.
    char *name;
    uint64_t count;
    uint64_t alloc_secs;
};
```

Standard TRES types:
- `TRES_CPU` - CPUs
- `TRES_MEM` - Memory
- `TRES_ENERGY` - Energy consumption
- `TRES_NODE` - Nodes
- `TRES_BILLING` - Billing units
- `TRES_FS_DISK` - Filesystem disk
- `TRES_VMEM` - Virtual memory
- `TRES_PAGES` - Memory pages

---

## Database Schema (MySQL)

### Core Tables

| Table | Purpose |
|-------|---------|
| `cluster_table` | Cluster definitions |
| `acct_table` | Account hierarchy |
| `user_table` | User records |
| `assoc_table` | User-account associations |
| `qos_table` | QoS definitions |
| `tres_table` | TRES type definitions |
| `wckey_table` | Workload characterization keys |

### Job Tables

| Table | Purpose |
|-------|---------|
| `job_table` | Job records |
| `step_table` | Job step records |
| `suspend_table` | Job suspension history |
| `resv_table` | Reservation records |

### Usage/Accounting Tables

| Table | Purpose |
|-------|---------|
| `usage_day_table` | Daily usage aggregation |
| `usage_hour_table` | Hourly usage aggregation |
| `usage_month_table` | Monthly usage aggregation |
| `cluster_usage_*_table` | Cluster-wide usage |
| `assoc_usage_*_table` | Per-association usage |

---

## Data Flow

```
Job Submission → job_record_t (slurmctld)
                      ↓
              Job Execution
                      ↓
              Job Completion
                      ↓
         slurmdb_job_rec_t (slurmdbd)
                      ↓
              MySQL Tables
```

---

## Key Relationships

```
job_record_t
├── job_details_t (constraints)
├── step_record_t[] (job steps)
├── job_resources_t (allocation)
├── job_array_struct_t (if array job)
├── part_record_t* → partition
├── slurmdb_assoc_rec_t* → association
└── slurmdb_qos_rec_t* → QoS

node_record_t
├── config_record_t* → hardware config
├── part_record_t*[] → partitions
└── gres_list → generic resources

slurmdb_assoc_rec_t
├── slurmdb_assoc_rec_t* → parent
├── slurmdb_qos_rec_t[] → QoS list
└── slurmdb_assoc_usage_t → usage
```

---

## Source Files

| Structure | Header | Implementation |
|-----------|--------|----------------|
| job_record_t | `src/common/job_record.h` | `src/slurmctld/job_mgr.c` |
| node_record_t | `src/common/node_conf.h` | `src/slurmctld/node_mgr.c` |
| part_record_t | `src/common/part_record.h` | `src/slurmctld/partition_mgr.c` |
| slurmdb_* | `slurm/slurmdb.h` | `src/database/*.c` |
