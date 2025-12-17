# Slurm 使用者指南

> 產生日期：2025-12-17 | 適用對象：一般使用者

本指南提供 Slurm 叢集的日常使用說明，包含作業提交、監控和資源管理。

---

## 目錄

1. [快速入門](#快速入門)
2. [基本概念](#基本概念)
3. [提交作業](#提交作業)
4. [監控作業](#監控作業)
5. [管理作業](#管理作業)
6. [資源請求](#資源請求)
7. [進階功能](#進階功能)
8. [常見問題](#常見問題)
9. [命令參考](#命令參考)

---

## 快速入門

### 30 秒入門

```bash
# 檢查叢集狀態
sinfo

# 提交批次作業
sbatch myjob.sh

# 檢查作業狀態
squeue -u $USER

# 取消作業
scancel <job_id>
```

### 最簡單的批次腳本

```bash
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=result.out

echo "Hello from Slurm!"
hostname
date
```

儲存為 `myjob.sh`，然後執行 `sbatch myjob.sh`。

---

## 基本概念

### 什麼是 Slurm？

Slurm 是一套叢集資源管理與作業排程系統。它負責：
- 管理叢集中的計算資源
- 排程使用者提交的作業
- 追蹤資源使用情況

### 核心術語

| 術語 | 說明 |
|------|------|
| **作業（Job）** | 提交給 Slurm 執行的工作單元 |
| **節點（Node）** | 叢集中的一台計算機 |
| **分割區（Partition）** | 節點的邏輯分組，類似佇列 |
| **任務（Task）** | 作業中的一個程序實例 |
| **步驟（Step）** | 作業中的一個執行階段 |
| **CPU** | 處理器核心 |
| **TRES** | 可追蹤資源（CPU、記憶體、GPU 等）|

### 作業類型

| 類型 | 命令 | 說明 |
|------|------|------|
| 批次作業 | `sbatch` | 非互動式，提交後自動執行 |
| 互動式作業 | `salloc` | 分配資源後產生互動式 shell |
| 直接執行 | `srun` | 直接在分配的資源上執行命令 |

---

## 提交作業

### 批次作業（sbatch）

最常用的作業提交方式。

**基本語法：**
```bash
sbatch [選項] script.sh
```

**範例批次腳本：**
```bash
#!/bin/bash
#SBATCH --job-name=myanalysis      # 作業名稱
#SBATCH --output=output_%j.log     # 標準輸出（%j = 作業 ID）
#SBATCH --error=error_%j.log       # 標準錯誤
#SBATCH --time=01:00:00            # 最大執行時間（時:分:秒）
#SBATCH --nodes=1                  # 節點數
#SBATCH --ntasks=4                 # 任務數
#SBATCH --cpus-per-task=2          # 每任務 CPU 數
#SBATCH --mem=8G                   # 總記憶體
#SBATCH --partition=compute        # 分割區

# 載入模組（如果需要）
module load gcc python

# 執行程式
python my_script.py
```

### 互動式作業（salloc）

取得資源後進入互動式 shell。

```bash
# 基本互動式分配
salloc --nodes=1 --time=1:00:00

# 分配後會進入新的 shell，可以執行命令
srun hostname
srun python interactive_script.py

# 完成後退出
exit
```

### 直接執行（srun）

在已分配的資源上直接執行命令。

```bash
# 直接執行（會自動請求資源）
srun --ntasks=4 ./my_mpi_program

# 在互動式會話中執行
salloc --nodes=2
srun --ntasks=8 ./parallel_program
```

### 提交選項

| 選項 | 縮寫 | 說明 | 範例 |
|------|------|------|------|
| `--job-name` | `-J` | 作業名稱 | `-J mytest` |
| `--output` | `-o` | 標準輸出檔案 | `-o out.%j` |
| `--error` | `-e` | 標準錯誤檔案 | `-e err.%j` |
| `--time` | `-t` | 時間限制 | `-t 2:00:00` |
| `--nodes` | `-N` | 節點數 | `-N 2` |
| `--ntasks` | `-n` | 任務數 | `-n 8` |
| `--cpus-per-task` | `-c` | 每任務 CPU | `-c 4` |
| `--mem` | | 總記憶體 | `--mem=16G` |
| `--mem-per-cpu` | | 每 CPU 記憶體 | `--mem-per-cpu=2G` |
| `--partition` | `-p` | 分割區 | `-p gpu` |
| `--account` | `-A` | 計費帳戶 | `-A myproject` |
| `--gres` | | 通用資源 | `--gres=gpu:2` |

---

## 監控作業

### 檢視作業佇列（squeue）

```bash
# 檢視自己的作業
squeue -u $USER

# 檢視所有作業
squeue

# 檢視特定分割區
squeue -p compute

# 詳細格式
squeue -u $USER -l

# 自訂輸出格式
squeue -u $USER -o "%.10i %.9P %.8j %.8u %.8T %.10M %.6D %R"
```

**作業狀態代碼：**

| 代碼 | 狀態 | 說明 |
|------|------|------|
| PD | PENDING | 等待資源 |
| R | RUNNING | 執行中 |
| CG | COMPLETING | 完成中 |
| CD | COMPLETED | 已完成 |
| F | FAILED | 執行失敗 |
| CA | CANCELLED | 已取消 |
| TO | TIMEOUT | 超時 |
| NF | NODE_FAIL | 節點故障 |

### 檢視叢集資訊（sinfo）

```bash
# 基本叢集資訊
sinfo

# 檢視節點詳情
sinfo -N -l

# 檢視特定分割區
sinfo -p compute

# 檢視可用資源
sinfo -o "%P %a %l %D %T %N"
```

**節點狀態：**

| 狀態 | 說明 |
|------|------|
| idle | 閒置，可使用 |
| alloc | 已分配，完全使用中 |
| mix | 部分使用 |
| drain | 管理員標記為不可用 |
| down | 離線 |

### 檢視作業詳情

```bash
# 檢視作業詳細資訊
scontrol show job <job_id>

# 檢視執行中作業的資源使用
sstat -j <job_id> --format=JobID,MaxRSS,MaxVMSize,AveRSS

# 檢視已完成作業的記錄
sacct -j <job_id>
sacct -j <job_id> --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
```

---

## 管理作業

### 取消作業（scancel）

```bash
# 取消單一作業
scancel <job_id>

# 取消所有自己的作業
scancel -u $USER

# 取消特定分割區的作業
scancel -u $USER -p compute

# 取消特定狀態的作業
scancel -u $USER -t PENDING
```

### 修改作業（scontrol）

```bash
# 修改等待中作業的時間限制
scontrol update jobid=<job_id> TimeLimit=4:00:00

# 暫停作業（需要管理員權限或設定允許）
scontrol hold <job_id>

# 恢復暫停的作業
scontrol release <job_id>
```

### 作業相依性

```bash
# 等待前一作業完成後執行
sbatch --dependency=afterok:<job_id> next_job.sh

# 等待前一作業結束（無論成功或失敗）
sbatch --dependency=afterany:<job_id> cleanup.sh

# 多個相依性
sbatch --dependency=afterok:<job1>:<job2> final.sh
```

---

## 資源請求

### CPU 和記憶體

```bash
#!/bin/bash
#SBATCH --ntasks=1              # 單一任務
#SBATCH --cpus-per-task=8       # 8 個 CPU 核心
#SBATCH --mem=32G               # 32GB 記憶體
```

### MPI 程式

```bash
#!/bin/bash
#SBATCH --nodes=4               # 4 個節點
#SBATCH --ntasks-per-node=16    # 每節點 16 個任務
#SBATCH --cpus-per-task=1       # 每任務 1 個 CPU

module load mpi
srun ./my_mpi_program
```

### GPU 作業

```bash
#!/bin/bash
#SBATCH --partition=gpu         # GPU 分割區
#SBATCH --gres=gpu:2            # 2 個 GPU
#SBATCH --cpus-per-task=8       # 配合 GPU 的 CPU
#SBATCH --mem=64G               # 記憶體

module load cuda
./my_gpu_program
```

### 陣列作業

批次執行多個相似作業：

```bash
#!/bin/bash
#SBATCH --job-name=array_job
#SBATCH --array=1-100           # 作業索引 1 到 100
#SBATCH --output=output_%A_%a.out  # %A=陣列作業ID, %a=任務索引

# 使用 $SLURM_ARRAY_TASK_ID 取得當前索引
echo "Processing task $SLURM_ARRAY_TASK_ID"
python process.py --input=data_${SLURM_ARRAY_TASK_ID}.txt
```

陣列作業選項：
```bash
#SBATCH --array=1-100           # 1, 2, 3, ..., 100
#SBATCH --array=1,3,5,7         # 特定索引
#SBATCH --array=1-100:5         # 步長 5: 1, 6, 11, ...
#SBATCH --array=1-100%10        # 最多同時執行 10 個
```

---

## 進階功能

### 環境變數

Slurm 設定的環境變數：

| 變數 | 說明 |
|------|------|
| `SLURM_JOB_ID` | 作業 ID |
| `SLURM_JOB_NAME` | 作業名稱 |
| `SLURM_SUBMIT_DIR` | 提交目錄 |
| `SLURM_JOB_NODELIST` | 分配的節點清單 |
| `SLURM_NTASKS` | 任務數 |
| `SLURM_CPUS_PER_TASK` | 每任務 CPU 數 |
| `SLURM_ARRAY_TASK_ID` | 陣列任務索引 |
| `SLURM_ARRAY_JOB_ID` | 陣列作業 ID |

### 電子郵件通知

```bash
#!/bin/bash
#SBATCH --mail-user=user@example.com
#SBATCH --mail-type=BEGIN,END,FAIL    # 開始、結束、失敗時通知
```

通知類型：`NONE`, `BEGIN`, `END`, `FAIL`, `REQUEUE`, `ALL`

### 優先級查看

```bash
# 檢視作業優先級因素
sprio -u $USER

# 詳細優先級資訊
sprio -l -u $USER
```

### 資源使用報告

```bash
# 檢視自己的使用記錄
sacct -u $USER --starttime=2025-01-01

# 詳細格式
sacct -u $USER --format=JobID,JobName,Partition,Elapsed,State,ExitCode,MaxRSS,MaxVMSize

# 最近一週的作業
sacct -u $USER --starttime=$(date -d '7 days ago' +%Y-%m-%d)
```

---

## 常見問題

### Q: 作業一直在 PENDING 狀態？

檢查原因：
```bash
squeue -u $USER -o "%.10i %.9P %.20j %.8u %.8T %.10M %.6D %R"
```

常見原因：
- `Resources` - 等待資源可用
- `Priority` - 優先級不夠高
- `QOSMaxJobsPerUserLimit` - 超過 QoS 限制
- `AssocMaxJobsLimit` - 超過帳戶限制

### Q: 如何估算需要的資源？

1. 先用小資源測試
2. 使用 `sacct` 檢視實際使用：
   ```bash
   sacct -j <job_id> --format=JobID,MaxRSS,MaxVMSize,Elapsed
   ```
3. 根據結果調整

### Q: 如何檢查叢集有哪些分割區？

```bash
sinfo -s
```

### Q: 作業執行超時怎麼辦？

1. 增加時間限制重新提交
2. 使用檢查點功能（如果程式支援）
3. 拆分成多個較小的作業

---

## 命令參考

### 常用命令速查

| 命令 | 用途 |
|------|------|
| `sbatch` | 提交批次作業 |
| `srun` | 執行平行命令 |
| `salloc` | 互動式資源分配 |
| `squeue` | 檢視作業佇列 |
| `sinfo` | 檢視叢集資訊 |
| `scancel` | 取消作業 |
| `scontrol` | 檢視/修改作業詳情 |
| `sacct` | 檢視記帳資料 |
| `sstat` | 檢視執行中作業統計 |
| `sprio` | 檢視優先級 |

### 取得更多幫助

```bash
# 檢視命令說明
man sbatch
man srun
man squeue

# 簡短幫助
sbatch --help
squeue --help

# 官方文件
# https://slurm.schedmd.com/quickstart.html
```

---

## 相關文件

- [專案概覽](./project-overview.md) - Slurm 系統概述
- [架構文件](./architecture.md) - 系統架構說明
- [API 契約](./api-contracts.md) - REST API 使用
