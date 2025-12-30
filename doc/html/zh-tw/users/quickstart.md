# Slurm 快速入門使用指南

---

## TL;DR

Slurm 是一個開源的叢集管理與作業排程系統。主要指令包括：`sinfo`（查看系統狀態）、`squeue`（查看作業佇列）、`srun`（即時執行作業）、`sbatch`（提交批次作業）、`salloc`（分配資源）、`scancel`（取消作業）。

---

## Translation（翻譯）

### 概述

Slurm 是一個開源、容錯且高度可擴展的叢集管理（cluster management）與作業排程系統（job scheduling system），適用於大型和小型 Linux 叢集。Slurm 的運作不需要修改核心（kernel），且相對自給自足。作為叢集工作負載管理器（cluster workload manager），Slurm 有三個關鍵功能：

1. **資源分配**：為使用者分配對資源（運算節點）的獨佔或非獨佔存取權，讓他們可以在一段時間內執行工作
2. **工作執行框架**：提供一個框架來啟動、執行和監控工作（通常是平行作業）
3. **資源仲裁**：透過管理待處理工作的佇列（queue）來仲裁資源競爭

### 架構

如圖 1 所示，Slurm 由以下元件組成：
- **slurmd** 常駐程式（daemon）：在每個運算節點上執行
- **slurmctld** 常駐程式：在管理節點上執行的中央控制器（可選配備援伺服器）

slurmd 常駐程式提供容錯的階層式通訊。

使用者指令包括：
- **sacct**：查詢作業帳務資訊
- **sacctmgr**：帳務管理
- **salloc**：即時分配資源
- **sattach**：附加到執行中的作業
- **sbatch**：提交批次作業
- **sbcast**：廣播檔案到節點
- **scancel**：取消作業
- **scontrol**：管理控制工具
- **scrontab**：排程週期性作業
- **sdiag**：診斷資訊
- **sh5util**：HDF5 工具
- **sinfo**：系統資訊
- **sprio**：優先權資訊
- **squeue**：作業佇列狀態
- **sreport**：報表工具
- **srun**：執行作業
- **sshare**：公平共享資訊
- **sstat**：作業統計
- **strigger**：觸發器管理
- **sview**：圖形介面

所有指令都可以在叢集中的任何地方執行。

### Slurm 管理的實體

Slurm 常駐程式管理的實體包括：

| 實體 | 說明 |
|------|------|
| **節點（nodes）** | Slurm 中的運算資源 |
| **分割區（partitions）** | 將節點分組為邏輯集合（可重疊） |
| **作業（jobs）** | 在指定時間內分配給使用者的資源 |
| **作業步驟（job steps）** | 作業內的（可能是平行的）任務集合 |

分割區可視為作業佇列，每個佇列都有各種限制條件，如作業大小限制、作業時間限制、允許使用的使用者等。優先權排序的作業會在分割區內分配節點，直到該分割區的資源（節點、處理器、記憶體等）耗盡。一旦作業被分配了一組節點，使用者就可以在分配範圍內以任何配置啟動作業步驟形式的平行工作。

### 指令說明

所有 Slurm 常駐程式、指令和 API 函式都有 man page。指令選項 `--help` 也提供選項的簡要摘要。請注意，指令選項都是區分大小寫的。

| 指令 | 功能說明 |
|------|----------|
| **sacct** | 報告作用中或已完成作業的帳務資訊 |
| **salloc** | 即時為作業分配資源，通常用於分配資源並啟動 shell |
| **sattach** | 將標準輸入、輸出、錯誤和訊號功能附加到執行中的作業 |
| **sbatch** | 提交作業腳本以供稍後執行 |
| **sbcast** | 將檔案從本地磁碟傳輸到分配給作業的節點的本地磁碟 |
| **scancel** | 取消待處理或執行中的作業或作業步驟 |
| **scontrol** | 用於查看和/或修改 Slurm 狀態的管理工具 |
| **sinfo** | 報告分割區和節點的狀態 |
| **sprio** | 顯示影響作業優先權的詳細元件 |
| **squeue** | 報告作業或作業步驟的狀態 |
| **srun** | 提交作業執行或即時啟動作業步驟 |
| **sshare** | 顯示叢集上公平共享使用的詳細資訊 |
| **sstat** | 取得執行中作業或作業步驟的資源使用資訊 |
| **strigger** | 設定、取得或查看事件觸發器 |
| **sview** | 圖形使用者介面，用於查看和更新作業、分割區和節點的狀態 |

### 最佳實務：大量作業

考慮將相關工作放入具有多個作業步驟的單一 Slurm 作業中，這樣做既可以提高效能，也便於管理。每個 Slurm 作業可以包含大量作業步驟，而 Slurm 管理作業步驟的開銷遠低於管理個別作業。

**作業陣列（Job arrays）** 是管理具有相同資源需求的批次作業集合的有效機制。大多數 Slurm 指令可以將作業陣列作為單獨元素（任務）或作為單一實體來管理。

### MPI 支援

MPI 的使用取決於所使用的 MPI 類型。這些不同的 MPI 實作使用三種根本不同的操作模式：

1. **Slurm 直接啟動**：Slurm 直接啟動任務並透過 PMI2 或 PMIx API 執行通訊初始化（大多數現代 MPI 實作支援）
2. **mpirun 使用 Slurm 基礎設施**：Slurm 為作業建立資源分配，然後 mpirun 使用 Slurm 的基礎設施啟動任務（舊版 OpenMPI）
3. **mpirun 使用其他機制**：Slurm 為作業建立資源分配，然後 mpirun 使用 Slurm 以外的機制（如 SSH 或 RSH）啟動任務

支援的 MPI 實作包括：Intel MPI、MPICH2、MVAPICH2、Open MPI。

---

## Explanation（解釋）

### 什麼是 Slurm？

想像你有一個大型電腦叢集，裡面有數百甚至數千台電腦。Slurm 就像是這個叢集的「交通警察」和「資源分配員」：

1. **資源管理**：追蹤哪些電腦（節點）是空閒的、哪些正在使用
2. **作業排程**：決定誰的程式什麼時候可以執行
3. **公平使用**：確保每個人都能公平地使用運算資源

### 核心概念

```
┌─────────────────────────────────────────────────┐
│                  Slurm 叢集                      │
│                                                 │
│  ┌─────────┐      ┌─────────────────────────┐   │
│  │slurmctld│      │     分割區 (Partition)    │   │
│  │ (控制器) │      │   ┌─────┬─────┬─────┐   │   │
│  └────┬────┘      │   │節點1│節點2│節點3│   │   │
│       │           │   └─────┴─────┴─────┘   │   │
│       ▼           └─────────────────────────┘   │
│  ┌─────────┐                                    │
│  │  佇列   │  作業1 → 作業2 → 作業3 → ...      │
│  └─────────┘                                    │
└─────────────────────────────────────────────────┘
```

### 三種作業提交方式

| 方式 | 指令 | 適用情境 |
|------|------|----------|
| 即時互動 | `srun` | 快速測試、除錯 |
| 批次提交 | `sbatch` | 長時間執行的作業 |
| 分配資源 | `salloc` | 需要互動式 shell 的情況 |

---

## Practical Example（實用範例）

### 範例 1：查看系統狀態

```bash
# 查看叢集的分割區和節點狀態
sinfo
```

輸出範例：
```
PARTITION AVAIL  TIMELIMIT NODES  STATE NODELIST
debug*       up      30:00     2  down* adev[1-2]
debug*       up      30:00     3   idle adev[3-5]
batch        up      30:00     3  down* adev[6,13,15]
batch        up      30:00     3  alloc adev[7-8,14]
batch        up      30:00     4   idle adev[9-12]
```

**逐行說明：**
- `PARTITION`：分割區名稱，`*` 表示預設分割區
- `AVAIL`：分割區狀態（up=可用）
- `TIMELIMIT`：最大作業時間限制
- `NODES`：節點數量
- `STATE`：節點狀態（idle=空閒, alloc=已分配, down=離線）
- `NODELIST`：節點清單（使用壓縮表示法）

### 範例 2：查看作業佇列

```bash
# 查看目前的作業狀態
squeue
```

輸出範例：
```
JOBID PARTITION  NAME  USER ST  TIME NODES NODELIST(REASON)
65646     batch  chem  mike  R 24:19     2 adev[7-8]
65647     batch   bio  joan  R  0:09     1 adev14
65648     batch  math  phil PD  0:00     6 (Resources)
```

**欄位說明：**
- `ST`：作業狀態（R=執行中, PD=等待中）
- `TIME`：已執行時間
- `NODELIST(REASON)`：執行的節點或等待原因

### 範例 3：使用 srun 即時執行

```bash
# 在 3 個節點上執行 hostname 指令，顯示任務編號
srun -N3 -l /bin/hostname
```

輸出：
```
0: adev3
1: adev4
2: adev5
```

**參數說明：**
- `-N3`：請求 3 個節點
- `-l`：在輸出前加上任務編號

### 範例 4：提交批次作業

```bash
# 建立作業腳本
cat > my_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=my_test      # 作業名稱
#SBATCH --nodes=2               # 請求 2 個節點
#SBATCH --ntasks=4              # 總共 4 個任務
#SBATCH --time=00:10:00         # 最大執行時間 10 分鐘
#SBATCH --output=result_%j.out  # 輸出檔案（%j 會被替換為作業 ID）

echo "作業開始於 $(date)"
echo "在節點 $(hostname) 上執行"

srun /bin/hostname
srun /bin/pwd

echo "作業結束於 $(date)"
EOF

# 提交作業
sbatch my_job.sh
```

### 範例 5：互動式分配資源

```bash
# 分配 2 個節點並進入互動式 shell
salloc -N2 bash

# 在分配的節點上執行指令
srun hostname

# 完成後退出
exit
```

### 範例 6：取消作業

```bash
# 提交作業
sbatch test.sh
# 輸出：Submitted batch job 473

# 查看作業狀態
squeue

# 取消作業
scancel 473
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| 未指定時間限制 | 作業可能使用預設的短時間限制而被終止 | 使用 `--time` 指定足夠的時間 |
| 請求過多資源 | 作業永遠在等待 | 檢查 `sinfo` 確認可用資源 |
| 忘記載入模組 | 程式找不到所需的函式庫 | 在腳本中加入 `module load` |
| 輸出檔案覆蓋 | 多次執行覆蓋結果 | 使用 `%j`（作業 ID）在檔名中 |

### ✅ 實用技巧

1. **查看作業為何等待**
   ```bash
   squeue --job=<jobid> --format="%r"
   ```

2. **查看詳細作業資訊**
   ```bash
   scontrol show job <jobid>
   ```

3. **估算作業開始時間**
   ```bash
   squeue --start --job=<jobid>
   ```

4. **使用作業陣列處理多個相似任務**
   ```bash
   sbatch --array=1-100 my_array_job.sh
   ```

5. **設定電子郵件通知**
   ```bash
   #SBATCH --mail-type=END,FAIL
   #SBATCH --mail-user=your@email.com
   ```

---

## Quick Reference（快速參考）

| 指令 | 功能 | 常用選項 |
|------|------|----------|
| `sinfo` | 查看系統狀態 | `-N`（按節點顯示）, `-p <partition>`（指定分割區） |
| `squeue` | 查看作業佇列 | `-u <user>`（指定使用者）, `-j <jobid>`（指定作業） |
| `srun` | 即時執行 | `-N`（節點數）, `-n`（任務數）, `-t`（時間限制） |
| `sbatch` | 提交批次作業 | `-J`（作業名稱）, `-o`（輸出檔）, `-e`（錯誤檔） |
| `salloc` | 分配資源 | `-N`（節點數）, `-t`（時間限制） |
| `scancel` | 取消作業 | `-u <user>`（取消使用者所有作業） |
| `scontrol` | 管理控制 | `show job`、`show node`、`show partition` |
| `sacct` | 作業帳務 | `-j <jobid>`（指定作業）, `--format`（輸出格式） |

### 作業狀態代碼

| 代碼 | 全名 | 說明 |
|------|------|------|
| `PD` | PENDING | 等待資源 |
| `R` | RUNNING | 執行中 |
| `CG` | COMPLETING | 完成中 |
| `CD` | COMPLETED | 已完成 |
| `F` | FAILED | 失敗 |
| `TO` | TIMEOUT | 超時 |
| `CA` | CANCELLED | 已取消 |

### 節點狀態

| 狀態 | 說明 |
|------|------|
| `idle` | 空閒，可接受作業 |
| `alloc` | 已分配給作業 |
| `mix` | 部分 CPU 已分配 |
| `down` | 離線或故障 |
| `drain` | 排空中，不接受新作業 |
