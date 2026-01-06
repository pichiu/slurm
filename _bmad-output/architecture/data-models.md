# Slurm 資料模型

> 產生日期：2025-12-17 | 來源：完整程式碼分析

## 概覽

Slurm 使用分層資料模型架構：
1. **執行時層**（slurmctld）- 活動作業、節點、分割區管理
2. **記帳層**（slurmdbd）- 歷史記錄、使用量追蹤
3. **持久層**（MySQL）- 長期資料儲存

---

## 核心執行時資料結構

### 1. job_record_t（作業記錄）

**位置**：`src/common/job_record.h`

slurmctld 中表示作業的主要資料結構。

```c
// 關鍵欄位（簡化版）
struct job_record {
    // 識別
    uint32_t job_id;
    uint32_t array_job_id;
    uint32_t array_task_id;
    uint32_t het_job_id;

    // 使用者/帳戶
    char *account;
    uint32_t user_id;
    char *user_name;
    uint32_t group_id;

    // 資源
    job_resources_t *job_resrcs;
    bitstr_t *node_bitmap;
    char *nodes;
    uint32_t node_cnt;
    uint32_t total_cpus;
    uint64_t pn_min_memory;

    // 時間
    time_t submit_time;
    time_t start_time;
    time_t end_time;
    time_t suspend_time;
    uint32_t time_limit;

    // 狀態
    uint32_t job_state;
    uint16_t state_reason;
    uint32_t priority;
    uint32_t exit_code;

    // 參照
    job_details_t *details;
    List step_list;
    part_record_t *part_ptr;
    slurmdb_assoc_rec_t *assoc_ptr;
    slurmdb_qos_rec_t *qos_ptr;
};
```

### 2. node_record_t（節點記錄）

**位置**：`src/common/node_conf.h`

表示叢集中的計算節點。

```c
struct node_record {
    // 識別
    char *name;
    char *comm_name;
    char *node_hostname;
    uint32_t index;

    // 硬體
    uint16_t cpus;
    uint16_t cpus_efctv;
    uint16_t cores;
    uint16_t threads;
    uint16_t tot_sockets;
    uint64_t real_memory;
    uint32_t tmp_disk;

    // 狀態
    uint32_t node_state;
    uint32_t next_state;
    char *reason;
    time_t reason_time;
    time_t last_response;

    // 負載
    float cpu_load;
    uint64_t free_mem;
    uint32_t up_time;

    // 資源
    List gres_list;
    uint64_t *tres_cnt;
    char *tres_str;

    // 分割區關聯
    part_record_t **part_pptr;
    config_record_t *config_ptr;
};
```

### 3. part_record_t（分割區記錄）

**位置**：`src/common/part_record.h`

表示叢集中的分割區（佇列）。

```c
struct part_record {
    // 識別
    char *name;
    char *nodes;
    bitstr_t *node_bitmap;

    // 容量
    uint32_t total_nodes;
    uint32_t total_cpus;
    uint32_t max_nodes;
    uint32_t min_nodes;

    // 限制
    uint32_t max_time;
    uint32_t default_time;
    uint32_t max_cpus_per_node;
    uint64_t max_mem_per_cpu;
    uint16_t max_share;

    // 優先級
    uint16_t priority_job_factor;
    uint16_t priority_tier;

    // 狀態
    uint16_t state_up;
    uint32_t flags;

    // QoS
    slurmdb_qos_rec_t *qos_ptr;
    char *billing_weights;
};
```

### 4. job_resources_t（作業資源分配）

**位置**：`src/common/job_resources.h`

追蹤作業的詳細資源分配。

```c
struct job_resources {
    // CPU 分配
    uint16_t *cpus;
    uint32_t cpu_array_cnt;
    uint16_t *cpu_array_value;
    uint32_t *cpu_array_reps;

    // 核心資訊
    bitstr_t *core_bitmap;
    uint16_t *cores_per_socket;
    uint16_t *sockets_per_node;

    // 記憶體
    uint64_t *memory_allocated;
    uint64_t *memory_used;

    // 節點分配
    bitstr_t *node_bitmap;
    char *nodes;
    uint32_t nhosts;
    uint32_t ncpus;

    // 使用量追蹤
    uint16_t *cpus_used;
    bitstr_t *core_bitmap_used;
};
```

---

## 記帳資料結構（slurmdb）

**位置**：`slurm/slurmdb.h`

### 1. slurmdb_assoc_rec_t（關聯記錄）

連結使用者、帳戶、叢集和分割區與資源限制。

```c
struct slurmdb_assoc_rec {
    // 識別
    uint32_t id;
    char *acct;
    char *user;
    char *cluster;
    char *partition;
    uid_t uid;

    // 階層
    char *parent_acct;
    uint32_t parent_id;
    char *lineage;

    // 群組限制
    uint32_t grp_jobs;
    uint32_t grp_submit_jobs;
    char *grp_tres;
    char *grp_tres_mins;
    char *grp_tres_run_mins;
    uint32_t grp_wall;

    // 個人限制
    uint32_t max_jobs;
    uint32_t max_submit_jobs;
    char *max_tres_pj;
    char *max_tres_run_mins;
    uint32_t max_wall_pj;

    // QoS
    uint32_t def_qos_id;
    List qos_list;

    // 使用量
    slurmdb_assoc_usage_t *usage;
    uint32_t shares_raw;
};
```

### 2. slurmdb_job_rec_t（資料庫作業記錄）

儲存在記帳資料庫中的歷史作業記錄。

```c
struct slurmdb_job_rec {
    // 識別
    uint32_t jobid;
    uint32_t array_job_id;
    uint32_t array_task_id;
    char *cluster;
    uint32_t het_job_id;

    // 帳戶資訊
    char *account;
    uid_t uid;
    char *user;
    uint32_t associd;

    // 時間戳記
    time_t submit;
    time_t start;
    time_t end;
    time_t eligible;

    // 資源
    uint32_t req_cpus;
    uint64_t req_mem;
    char *tres_alloc_str;
    char *tres_req_str;

    // 狀態
    uint32_t state;
    int32_t exitcode;
    int32_t derived_ec;

    // CPU 時間
    uint32_t user_cpu_sec;
    uint32_t sys_cpu_sec;
    uint32_t tot_cpu_sec;

    // 分配
    char *alloc_nodes;
    char *nodes;
    char *partition;

    // 步驟
    List steps;
};
```

### 3. slurmdb_qos_rec_t（QoS 記錄）

服務品質設定。

```c
struct slurmdb_qos_rec {
    // 識別
    uint32_t id;
    char *name;
    char *description;

    // 優先級
    uint32_t priority;
    double norm_priority;
    uint16_t preempt_mode;
    double usage_factor;

    // 群組限制
    uint32_t grp_jobs;
    uint32_t grp_submit_jobs;
    char *grp_tres;
    char *grp_tres_mins;
    char *grp_tres_run_mins;
    uint32_t grp_wall;

    // 每帳戶/使用者限制
    uint32_t max_jobs_pa;
    uint32_t max_jobs_pu;
    uint32_t max_submit_jobs_pa;
    uint32_t max_submit_jobs_pu;

    // 每作業限制
    char *max_tres_pj;
    char *max_tres_mins_pj;
    uint32_t max_wall_pj;

    // 搶佔
    List preempt_list;
    bitstr_t *preempt_bitstr;

    // 使用量
    slurmdb_qos_usage_t *usage;
};
```

### 4. slurmdb_tres_rec_t（TRES 記錄）

可追蹤資源類型。

```c
struct slurmdb_tres_rec {
    uint32_t id;
    char *type;    // cpu、mem、energy、node、billing 等
    char *name;
    uint64_t count;
    uint64_t alloc_secs;
};
```

標準 TRES 類型：
- `TRES_CPU` - CPU
- `TRES_MEM` - 記憶體
- `TRES_ENERGY` - 能源消耗
- `TRES_NODE` - 節點
- `TRES_BILLING` - 計費單位
- `TRES_FS_DISK` - 檔案系統磁碟
- `TRES_VMEM` - 虛擬記憶體
- `TRES_PAGES` - 記憶體頁面

---

## 資料庫綱要（MySQL）

### 核心資料表

| 資料表 | 用途 |
|--------|------|
| `cluster_table` | 叢集定義 |
| `acct_table` | 帳戶階層 |
| `user_table` | 使用者記錄 |
| `assoc_table` | 使用者-帳戶關聯 |
| `qos_table` | QoS 定義 |
| `tres_table` | TRES 類型定義 |
| `wckey_table` | 工作負載特徵鍵 |

### 作業資料表

| 資料表 | 用途 |
|--------|------|
| `job_table` | 作業記錄 |
| `step_table` | 作業步驟記錄 |
| `suspend_table` | 作業暫停歷史 |
| `resv_table` | 保留記錄 |

### 使用量/記帳資料表

| 資料表 | 用途 |
|--------|------|
| `usage_day_table` | 每日使用量聚合 |
| `usage_hour_table` | 每小時使用量聚合 |
| `usage_month_table` | 每月使用量聚合 |
| `cluster_usage_*_table` | 叢集範圍使用量 |
| `assoc_usage_*_table` | 每關聯使用量 |

---

## 資料流程

```
作業提交 → job_record_t（slurmctld）
                      ↓
              作業執行
                      ↓
              作業完成
                      ↓
         slurmdb_job_rec_t（slurmdbd）
                      ↓
              MySQL 資料表
```

---

## 主要關係

```
job_record_t
├── job_details_t（限制條件）
├── step_record_t[]（作業步驟）
├── job_resources_t（分配）
├── job_array_struct_t（如果是陣列作業）
├── part_record_t* → 分割區
├── slurmdb_assoc_rec_t* → 關聯
└── slurmdb_qos_rec_t* → QoS

node_record_t
├── config_record_t* → 硬體設定
├── part_record_t*[] → 分割區
└── gres_list → 通用資源

slurmdb_assoc_rec_t
├── slurmdb_assoc_rec_t* → 父階層
├── slurmdb_qos_rec_t[] → QoS 清單
└── slurmdb_assoc_usage_t → 使用量
```

---

## 原始碼檔案

| 結構 | 標頭檔 | 實作 |
|------|--------|------|
| job_record_t | `src/common/job_record.h` | `src/slurmctld/job_mgr.c` |
| node_record_t | `src/common/node_conf.h` | `src/slurmctld/node_mgr.c` |
| part_record_t | `src/common/part_record.h` | `src/slurmctld/partition_mgr.c` |
| slurmdb_* | `slurm/slurmdb.h` | `src/database/*.c` |
