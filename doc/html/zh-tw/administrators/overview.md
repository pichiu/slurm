# Slurm 概述

---

## TL;DR

Slurm 是一個開源的叢集工作負載管理器，具備三大核心功能：資源分配、作業執行監控、作業佇列管理。系統架構包含 `slurmctld`（控制器）、`slurmd`（節點常駐程式）、`slurmdbd`（資料庫常駐程式）和 `slurmrestd`（REST API）。Slurm 採用模組化外掛架構，可彈性擴充功能。

---

## Translation（翻譯）

### 概述

Slurm 是一個開源、容錯且高度可擴展的叢集管理（cluster management）與作業排程系統（job scheduling system），適用於大型和小型 Linux 叢集。Slurm 的運作不需要修改核心（kernel），且相對自給自足。作為叢集工作負載管理器（cluster workload manager），Slurm 有三個關鍵功能：

1. **資源分配**：為使用者分配對資源（運算節點）的獨佔或非獨佔存取權，讓他們可以在一段時間內執行工作
2. **工作執行框架**：提供一個框架來啟動、執行和監控工作（通常是平行作業）
3. **資源仲裁**：透過管理待處理工作的佇列來仲裁資源競爭

可選的外掛程式（plugins）可用於：
- [帳務管理](accounting.md)
- [進階預約](reservations.md)
- [分時排程](gang_scheduling.md)（用於平行作業的時間共享）
- 回填排程（backfill scheduling）
- [拓撲最佳化資源選擇](topology.md)
- [資源限制](resource_limits.md)（依使用者或帳戶）
- 複雜的[多因子作業優先權](priority_multifactor.md)演算法

### 架構

Slurm 有一個集中式管理器 **slurmctld**，用於監控資源和工作。可能還有一個備援管理器，在發生故障時接管這些職責。每台運算伺服器（節點）都有一個 **slurmd** 常駐程式（daemon），可以比作遠端 shell：它等待工作、執行工作、回傳狀態，然後等待更多工作。slurmd 常駐程式提供容錯的階層式通訊。

還有一個可選的 **slurmdbd**（Slurm 資料庫常駐程式），可用於在單一資料庫中記錄多個 Slurm 管理叢集的帳務資訊。

還有一個可選的 **[slurmrestd](rest.md)**（Slurm REST API 常駐程式），可用於透過 [REST API](https://en.wikipedia.org/wiki/Representational_state_transfer) 與 Slurm 互動。

**使用者工具包括：**
- **srun**：啟動作業
- **scancel**：終止佇列中或執行中的作業
- **sinfo**：報告系統狀態
- **squeue**：報告作業狀態
- **sacct**：取得執行中或已完成作業的資訊

**sview** 指令以圖形方式報告系統和作業狀態，包括網路拓撲。

**管理工具：**
- **scontrol**：用於監控和/或修改叢集的配置和狀態資訊
- **sacctmgr**：用於管理資料庫，可識別叢集、有效使用者、有效帳戶等

所有功能都有 API 可用。

### 外掛機制

Slurm 有一個通用的外掛機制，可輕鬆支援各種基礎設施。這允許使用積木式方法進行各種 Slurm 配置。目前的外掛包括：

| 外掛類型 | 功能說明 |
|----------|----------|
| **帳務儲存（Accounting Storage）** | 主要用於儲存作業的歷史資料。與 SlurmDBD 一起使用時，還可以提供基於限制的系統和歷史系統狀態 |
| **帳戶能源收集（Account Gather Energy）** | 收集每個作業或系統節點的能源消耗資料。此外掛與帳務儲存和作業帳戶收集外掛整合 |
| **通訊認證（Authentication）** | 提供 Slurm 各元件之間的認證機制 |
| **[容器](containers.md)（Containers）** | HPC 工作負載容器支援和實作 |
| **憑證（Credential）** | 用於產生數位簽章的機制，用於驗證作業步驟是否被授權在特定節點上執行 |
| **[通用資源](gres.md)（Generic Resources）** | 提供控制通用資源（包括 GPU）的介面 |
| **[作業提交](job_submit_plugins.html)（Job Submit）** | 允許站點特定控制提交和更新時的作業需求 |
| **作業帳務收集（Job Accounting Gather）** | 收集作業步驟資源使用資料 |
| **作業完成記錄（Job Completion Logging）** | 記錄作業的終止資料 |
| **啟動器（Launchers）** | 控制 srun 指令啟動任務的機制 |
| **MPI** | 為各種 MPI 實作提供不同的鉤子，例如設定 MPI 特定的環境變數 |
| **[搶佔](preempt.md)（Preempt）** | 決定哪些作業可以搶佔其他作業以及要使用的搶佔機制 |
| **優先權（Priority）** | 在提交時和持續基礎上（例如隨著作業老化）為作業分配優先權 |
| **程序追蹤（Process tracking）** | 提供識別與每個作業相關程序的機制，用於作業帳務和訊號傳遞 |
| **排程器（Scheduler）** | 決定 Slurm 如何以及何時排程作業 |
| **節點選擇（Node selection）** | 用於決定作業分配所使用的資源 |
| **[站點因子](site_factor.html)（Site Factor）** | 在提交時和持續基礎上為作業分配多因子優先權的特定 site_factor 元件 |
| **交換器或互連（Switch）** | 與交換器或互連介面的外掛。對於大多數系統（乙太網路或 InfiniBand）不需要 |
| **任務親和性（Task Affinity）** | 提供將作業及其個別任務綁定到特定處理器的機制 |
| **網路拓撲（Network Topology）** | 基於網路拓撲最佳化資源選擇，用於作業分配和進階預約 |

### Slurm 管理的實體

如圖 2 所示，這些 Slurm 常駐程式管理的實體包括：

| 實體 | 說明 |
|------|------|
| **節點（nodes）** | Slurm 中的運算資源 |
| **分割區（partitions）** | 將節點分組為邏輯集合 |
| **作業（jobs）** | 在指定時間內分配給使用者的資源 |
| **作業步驟（job steps）** | 作業內的（可能是平行的）任務集合 |

分割區可視為作業佇列（job queues），每個佇列都有各種限制條件，如作業大小限制、作業時間限制、允許使用的使用者等。優先權排序的作業會在分割區內分配節點，直到該分割區的資源（節點、處理器、記憶體等）耗盡。一旦作業被分配了一組節點，使用者就可以在分配範圍內以任何配置啟動作業步驟形式的平行工作。例如，可以啟動一個利用所有分配給作業的節點的單一作業步驟，或者多個作業步驟可以獨立使用分配的一部分。

Slurm 為分配給作業的處理器提供資源管理，因此可以同時提交和排隊多個作業步驟，直到作業分配範圍內有可用資源。

### 可配置性

監控的節點狀態包括：處理器數量、實體記憶體大小、暫存磁碟空間大小和狀態（UP、DOWN 等）。額外的節點資訊包括權重（被分配工作的優先順序）和特性（任意資訊，如處理器速度或類型）。

節點被分組到分割區中，這些分割區可能包含重疊的節點，因此最好將它們視為作業佇列。

**分割區資訊包括：**
- 名稱
- 相關節點列表
- 狀態（UP 或 DOWN）
- 最大作業時間限制
- 每個作業的最大節點數
- 群組存取列表
- 優先權（在節點屬於多個分割區時很重要）
- 共享節點存取政策，可選的過度訂閱等級（例如 YES、NO 或 FORCE:2）

使用位元圖來表示節點，排程決策可以透過執行少量比較和一系列快速位元圖操作來做出。

**範例配置檔（部分）：**

```bash
#
# 範例 /etc/slurm.conf
#
SlurmctldHost=linux0001  # 主要伺服器
SlurmctldHost=linux0002  # 備援伺服器
#
AuthType=auth/munge
Epilog=/usr/local/slurm/sbin/epilog
PluginDir=/usr/local/slurm/lib
Prolog=/usr/local/slurm/sbin/prolog
SlurmctldPort=7002
SlurmctldTimeout=120
SlurmdPort=7003
SlurmdSpoolDir=/var/tmp/slurmd.spool
SlurmdTimeout=120
StateSaveLocation=/usr/local/slurm/slurm.state
TmpFS=/tmp
#
# 節點配置
#
NodeName=DEFAULT CPUs=4 TmpDisk=16384 State=IDLE
NodeName=lx[0001-0002] State=DRAINED
NodeName=lx[0003-8000] RealMemory=2048 Weight=2
NodeName=lx[8001-9999] RealMemory=4096 Weight=6 Feature=video
#
# 分割區配置
#
PartitionName=DEFAULT MaxTime=30 MaxNodes=2
PartitionName=login Nodes=lx[0001-0002] State=DOWN
PartitionName=debug Nodes=lx[0003-0030] State=UP Default=YES
PartitionName=class Nodes=lx[0031-0040] AllowGroups=students
PartitionName=DEFAULT MaxTime=UNLIMITED MaxNodes=4096
PartitionName=batch Nodes=lx[0041-9999]
```

---

## Explanation（解釋）

### Slurm 的核心架構

```
┌────────────────────────────────────────────────────────────────┐
│                        Slurm 叢集架構                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     │
│   │  使用者工具  │     │  管理工具    │     │  REST API   │     │
│   │ srun,sbatch │     │  scontrol   │     │  slurmrestd │     │
│   │ squeue,sinfo│     │  sacctmgr   │     │             │     │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘     │
│          │                   │                   │             │
│          ▼                   ▼                   ▼             │
│   ┌─────────────────────────────────────────────────────┐     │
│   │              slurmctld (控制器常駐程式)               │     │
│   │         主要控制器 + 可選備援控制器                    │     │
│   └──────────────────────┬──────────────────────────────┘     │
│                          │                                     │
│          ┌───────────────┼───────────────┐                    │
│          ▼               ▼               ▼                    │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│   │  slurmd  │    │  slurmd  │    │  slurmd  │  ...          │
│   │  節點 1  │    │  節點 2  │    │  節點 3  │               │
│   └──────────┘    └──────────┘    └──────────┘               │
│                                                                │
│   ┌─────────────────────────────────────────────────────┐     │
│   │           slurmdbd (資料庫常駐程式，可選)             │     │
│   │              儲存帳務資訊到資料庫                      │     │
│   └─────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

### 四大核心實體的關係

```
叢集 (Cluster)
    │
    ├── 分割區 (Partition) - 如同作業佇列
    │       │
    │       ├── 節點 (Node) - 實際的運算資源
    │       │       ├── CPU
    │       │       ├── 記憶體
    │       │       └── GPU (透過 GRES)
    │       │
    │       └── 限制條件
    │               ├── 最大時間
    │               ├── 最大節點數
    │               └── 允許使用者/群組
    │
    └── 作業 (Job)
            │
            ├── 作業步驟 (Job Step) 1
            │       └── 任務 (Tasks)
            │
            └── 作業步驟 (Job Step) 2
                    └── 任務 (Tasks)
```

### 外掛架構的優點

Slurm 的模組化設計讓您可以：

1. **依需求選擇元件**：不需要的功能可以不安裝
2. **輕鬆擴充**：新增功能只需安裝對應外掛
3. **客製化**：可以開發自己的外掛來符合特定需求
4. **升級彈性**：可以單獨升級特定外掛

---

## Practical Example（實用範例）

### 範例 1：查看叢集架構資訊

```bash
# 查看控制器配置
scontrol show config | grep -E "SlurmctldHost|ClusterName"
```

輸出範例：
```
ClusterName             = mycluster
SlurmctldHost           = linux0001
SlurmctldHost           = linux0002
```

**說明：**
- `ClusterName`：叢集名稱
- `SlurmctldHost`：列出主要和備援控制器

### 範例 2：查看節點詳細資訊

```bash
# 查看特定節點的詳細配置
scontrol show node node001
```

輸出範例：
```
NodeName=node001 Arch=x86_64 CoresPerSocket=8
   CPUAlloc=0 CPUTot=16 CPULoad=0.01
   AvailableFeatures=intel,gpu
   ActiveFeatures=intel,gpu
   Gres=gpu:2
   NodeAddr=192.168.1.101 NodeHostName=node001
   RealMemory=64000 AllocMem=0 FreeMem=62000 Sockets=2
   State=IDLE ThreadsPerCore=1 TmpDisk=100000
   Weight=1 Owner=N/A MCS_label=N/A
   Partitions=batch,gpu
```

**欄位說明：**
- `CPUTot`：總 CPU 核心數
- `RealMemory`：實體記憶體（MB）
- `Gres`：通用資源（如 GPU）
- `State`：節點狀態
- `Partitions`：節點所屬的分割區

### 範例 3：查看分割區配置

```bash
# 查看所有分割區的詳細配置
scontrol show partition
```

輸出範例：
```
PartitionName=debug
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=YES QoS=N/A
   DefaultTime=00:30:00 DisableRootJobs=NO ExclusiveUser=NO
   MaxNodes=4 MaxTime=01:00:00 MinNodes=0
   Nodes=node[001-010]
   State=UP TotalCPUs=160 TotalNodes=10

PartitionName=batch
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=NO QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO
   MaxNodes=UNLIMITED MaxTime=7-00:00:00 MinNodes=0
   Nodes=node[011-100]
   State=UP TotalCPUs=1440 TotalNodes=90
```

### 範例 4：查看 Slurm 常駐程式狀態

```bash
# 查看控制器狀態
scontrol ping
```

輸出範例：
```
Slurmctld(primary) at linux0001 is UP
Slurmctld(backup) at linux0002 is UP
```

### 範例 5：查看已安裝的外掛

```bash
# 查看目前使用的外掛配置
scontrol show config | grep -E "Plugin|Type"
```

輸出範例：
```
AccountingStorageType   = accounting_storage/slurmdbd
AuthType                = auth/munge
CredType                = cred/munge
JobAcctGatherType       = jobacct_gather/cgroup
MpiDefault              = none
PreemptType             = preempt/partition_prio
PriorityType            = priority/multifactor
ProctrackType           = proctrack/cgroup
SchedulerType           = sched/backfill
SelectType              = select/cons_tres
SwitchType              = switch/none
TaskPlugin              = task/cgroup,task/affinity
TopologyPlugin          = topology/none
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| 節點重疊配置錯誤 | 同一節點在多個分割區的權重設定衝突 | 使用 `scontrol show partition` 檢查配置 |
| 未設定備援控制器 | 主控制器故障時叢集無法運作 | 配置 `SlurmctldHost` 備援伺服器 |
| 外掛不相容 | 升級後外掛版本不匹配 | 確保所有外掛與 Slurm 版本相容 |
| 帳務資料庫未連線 | `sacct` 無法查詢歷史資料 | 檢查 `slurmdbd` 是否正常運作 |

### ✅ 實用技巧

1. **快速檢查叢集健康狀態**
   ```bash
   sinfo -R  # 查看節點離線原因
   scontrol ping  # 檢查控制器狀態
   ```

2. **查看完整配置**
   ```bash
   scontrol show config  # 顯示所有配置參數
   ```

3. **檢查資料庫連線**
   ```bash
   sacctmgr show cluster  # 列出已註冊的叢集
   ```

4. **驗證節點狀態**
   ```bash
   sinfo -N -l  # 以長格式顯示所有節點
   ```

5. **監控系統效能**
   ```bash
   sdiag  # 顯示排程器診斷資訊
   ```

---

## Quick Reference（快速參考）

### 架構元件

| 元件 | 功能 | 預設埠號 |
|------|------|----------|
| `slurmctld` | 中央控制器 | 6817 |
| `slurmd` | 節點常駐程式 | 6818 |
| `slurmdbd` | 資料庫常駐程式 | 6819 |
| `slurmrestd` | REST API | 依配置 |

### 主要配置檔

| 檔案 | 用途 |
|------|------|
| `/etc/slurm/slurm.conf` | 主要配置檔 |
| `/etc/slurm/slurmdbd.conf` | 資料庫配置 |
| `/etc/slurm/gres.conf` | 通用資源配置 |
| `/etc/slurm/topology.conf` | 網路拓撲配置 |

### 系統狀態指令

| 指令 | 功能 |
|------|------|
| `scontrol show config` | 顯示完整配置 |
| `scontrol show node` | 顯示節點資訊 |
| `scontrol show partition` | 顯示分割區資訊 |
| `scontrol ping` | 檢查控制器狀態 |
| `sinfo -R` | 顯示節點離線原因 |
| `sdiag` | 顯示排程診斷資訊 |

### 節點狀態代碼

| 狀態 | 說明 |
|------|------|
| `IDLE` | 空閒，可接受作業 |
| `ALLOCATED` | 已完全分配 |
| `MIXED` | 部分分配 |
| `DOWN` | 離線 |
| `DRAINED` | 排空中 |
| `DRAINING` | 正在排空 |
| `RESERVED` | 已預約 |
