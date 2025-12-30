# Slurm 異質作業支援

## TL;DR
- **異質作業 (Heterogeneous Job)** 允許單一作業包含多個具有不同資源需求的組件
- 使用 `:` 分隔不同組件的資源規格：`sbatch -N1 --mem=64G : -N4 --ntasks=32 job.sh`
- 每個組件有獨立的 job_id，但共享同一個 het_job_id（異質作業領導者 ID）
- 使用 `--het-group=` 選項指定在哪個組件上啟動應用程式
- 所有組件會同時排程，要麼全部開始，要麼全部等待

---

## Translation

### 概述

Slurm 17.11 及更高版本支援提交和管理異質作業 (Heterogeneous Jobs)，其中每個組件幾乎具有所有可用的作業選項，包括分割區、帳號和 QOS（服務品質）。例如，作業的一部分可能需要 128 個任務各有四個核心和 4 GB 記憶體，而作業的另一部分需要 16 GB 記憶體和一個 CPU。

### 提交作業

`salloc`、`sbatch` 和 `srun` 指令都可用於提交異質作業。異質作業各組件的資源規格應使用 `:` 字元分隔。例如：

```bash
$ sbatch --cpus-per-task=4 --mem-per-cpu=1  --ntasks=128 : \
         --cpus-per-task=1 --mem-per-cpu=16 --ntasks=1 my.bash
```

為異質作業（或作業步驟）的一個組件指定的選項將用於後續組件，在預期有幫助的範圍內。可以根據需要為每個組件重設傳播的選項（例如，可以為每個異質作業組件指定不同的帳號名稱）。例如，`--immediate` 和 `--job-name` 會傳播，而 `--ntasks` 和 `--mem-per-cpu` 會為每個組件重設為預設值。

#### 傳播的選項列表

- --account、--begin、--clusters、--comment
- --deadline、--delay-boot、--dependency、--distribution
- --error、--export、--exclude、--get-user-env
- --hold、--immediate、--input、--job-name
- --mem、--nice、--no-kill、--no-requeue
- --output、--priority、--profile、--qos
- --quiet、--reservation、--requeue、--signal
- --time、--time-min、--verbose、--wait-all-nodes
- --wckey、--workdir

批次作業選項可以包含在提交的腳本中，用於多個異質作業組件。每個組件應使用包含 `#SBATCH hetjob` 行的行分隔：

```bash
$ cat new.bash
#!/bin/bash
#SBATCH --cpus-per-task=4 --mem-per-cpu=16g --ntasks=1
#SBATCH hetjob
#SBATCH --cpus-per-task=2 --mem-per-cpu=1g  --ntasks=8

srun run.app

$ sbatch new.bash
```

這等同於：

```bash
$ cat my.bash
#!/bin/bash
srun run.app

$ sbatch --cpus-per-task=4 --mem-per-cpu=16g --ntasks=1 : \
         --cpus-per-task=2 --mem-per-cpu=1g  --ntasks=8 my.bash
```

批次腳本將在異質作業第一個組件的第一個節點上執行。

### 突發緩衝區

突發緩衝區可以是持久的或連結到特定作業 ID。由於異質作業由多個作業 ID 組成，作業特定的突發緩衝區只會與一個異質作業組件關聯。只有持久突發緩衝區才能被異質作業的所有組件存取。

### 管理作業

Slurm 為異質作業維護的資訊包括：

| 欄位 | 說明 |
|------|------|
| job_id | 異質作業的每個組件都有自己唯一的 job_id |
| het_job_id | 此識別號適用於異質作業的所有組件，等於第一個組件的 job_id（稱為「異質作業領導者」）|
| het_job_id_set | 識別與作業關聯的所有 job_id 值的正則表達式 |
| het_job_offset | 應用於異質作業每個組件的唯一序號，從 0 開始 |

**範例作業 ID：**

| job_id | het_job_id | het_job_offset | het_job_id_set |
|--------|------------|----------------|----------------|
| 123 | 123 | 0 | 123-127 |
| 124 | 123 | 1 | 123-127 |
| 125 | 123 | 2 | 123-127 |
| 126 | 123 | 3 | 123-127 |
| 127 | 123 | 4 | 123-127 |

`squeue` 和 `sview` 指令使用 `<het_job_id>+<het_job_offset>` 格式報告異質作業的組件。例如「123+4」表示異質作業 ID 123 及其第五個組件（注意：第一個組件的 het_job_offset 值為 0）。

```bash
$ squeue --job=93
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
 93+0     debug  bash  adam  R 18:18      1 nid00001
 93+1     debug  bash  adam  R 18:18      1 nid00011
 93+2     debug  bash  adam  R 18:18      1 nid00021
```

#### 取消作業

取消或發送訊號給異質作業領導者的請求將應用於該異質作業的所有組件。使用「#+#」表示法取消特定組件的請求僅適用於該特定組件：

```bash
$ scancel 93+1    # 只取消組件 1
$ scancel 93      # 取消所有組件
```

當異質作業處於待處理狀態時，只能取消整個作業而不是其個別組件。

#### 修改作業

使用 scontrol 修改單個組件必須使用「#+#」表示法指定作業 ID。指定 het_job_id 的請求將修改異質作業的所有組件：

```bash
# 更改異質作業 123 組件 2 的帳號：
$ scontrol update jobid=123+2 account=abc

# 更改異質作業 123 所有組件的時間限制：
$ scontrol update jobid=123 timelimit=60
```

以下操作只能針對異質作業領導者請求，並將應用於該異質作業的所有組件：
- requeue（重新排隊）
- resume（恢復）
- suspend（暫停）

#### sbcast 指令

sbcast 指令支援異質作業分配。預設情況下，sbcast 會將檔案複製到作業分配中的所有節點。-j/--jobid 選項可用於將檔案複製到個別組件：

```bash
$ sbcast --jobid=123   data /tmp/data   # 複製到所有組件
$ sbcast --jobid=123.0 app0 /tmp/app0   # 只複製到組件 0
$ sbcast --jobid=123.1 app1 /tmp/app1   # 只複製到組件 1
```

### 會計

Slurm 的會計資料庫記錄 het_job_id 和 het_job_offset 欄位。sacct 指令使用 `<het_job_id>+<het_job_offset>` 格式報告作業，並可接受相同格式的作業 ID 規格進行過濾：

```bash
$ sacct -j 67767
  JobID JobName Partition Account AllocCPUS     State ExitCode
------- ------- --------- ------- --------- --------- --------
67767+0     foo     debug    test         2 COMPLETED      0:0
67767+1     foo     debug    test         4 COMPLETED      0:0

$ sacct -j 67767+1
  JobID JobName Partition Account AllocCPUS     State ExitCode
------- ------- --------- ------- --------- --------- --------
67767+1     foo     debug    test         4 COMPLETED      0:0
```

### 啟動應用程式（作業步驟）

srun 指令用於啟動應用程式。預設情況下，應用程式僅在異質作業的第一個組件上啟動，但有選項可支援不同的行為。

#### --het-group 選項

srun 的 `--het-group` 選項定義要為哪些異質作業組件啟動應用程式。--het-group 選項接受定義要為單次 srun 執行啟動應用程式的組件的表達式。表達式可以在逗號分隔列表中包含一個或多個組件索引值。索引值範圍可以用連字號分隔的列表指定。預設情況下，應用程式僅在組件編號零上啟動。一些範例：

- `--het-group=2`
- `--het-group=0,4`
- `--het-group=1,3-5`

**重要**：跨多個作業分配執行單一應用程式的能力不適用於所有 MPI 實作或 Slurm MPI 外掛程式。可以透過在 Slurm 的 SchedulerParameters 配置參數中添加 `disable_hetjob_steps` 來禁用整個叢集的此功能。

**重要**：雖然 srun 指令可用於啟動異質作業步驟，但 mpirun 需要大量修改才能支援異質應用程式。

預設情況下，由單次 srun 執行啟動的應用程式（即使是異質作業的不同組件）會合併成一個具有不重疊任務 ID 的 MPI_COMM_WORLD：

```bash
$ srun --label -n2 : -n1 hostname
0: nid00012
1: nid00012
2: nid00013
```

所有作業步驟組件將具有相同的步驟 ID 值：

```bash
$ salloc -n1 : -n2 bash
salloc: Granted job allocation 1721
$ srun --het-group=0,1 true   # 啟動步驟 1721.0 和 1722.0
$ srun --het-group=0   true   # 啟動步驟 1721.1，沒有 1722.1
$ srun --het-group=0,1 true   # 啟動步驟 1721.2 和 1722.2
```

### 環境變數

Slurm 環境變數將透過在通常名稱後附加 `_HET_GROUP_` 和序號來為作業的每個組件獨立設定。此外，`SLURM_JOB_ID` 環境變數將包含異質作業領導者的作業 ID，`SLURM_HET_SIZE` 將包含作業中的組件數：

```bash
$ salloc -N1 : -N2 bash
salloc: Granted job allocation 11741
$ env | grep SLURM
SLURM_JOB_ID=11741
SLURM_HET_SIZE=2
SLURM_JOB_ID_HET_GROUP_0=11741
SLURM_JOB_ID_HET_GROUP_1=11742
SLURM_JOB_NODES_HET_GROUP_0=1
SLURM_JOB_NODES_HET_GROUP_1=2
SLURM_JOB_NODELIST_HET_GROUP_0=nid00001
SLURM_JOB_NODELIST_HET_GROUP_1=nid[00011-00012]
```

### 限制

- 回填排程器在追蹤未來 CPU 和記憶體使用方面有限制
- 在叢集聯邦中，異質作業將完全在提交作業的叢集上執行
- 磁性預約不能「吸引」異質作業
- 不支援異質作業的作業陣列
- srun 的 --no-allocate 選項不支援異質作業
- 單次 srun 每個異質作業組件只能啟動一個作業步驟
- sattach 一次只能附加到異質作業的單個組件
- 授權請求只允許在第一個組件作業上
- 異質作業僅由回填排程器外掛程式排程
- 不支援幫派排程操作

### 異質步驟

Slurm 20.11 版本引入了從非同質作業分配中請求異質作業步驟的能力。這允許您靈活地為作業步驟設定不同的佈局，而無需使用異質作業：

```bash
$ salloc -N2 --exclusive --gpus=10
salloc: Granted job allocation 61034
$ srun -N1 -n4 --gpus=4 printenv SLURMD_NODENAME : -N1 -n1 --gpus=6 printenv SLURMD_NODENAME
node02
node01
node01
node01
node01
```

---

## Explanation

### 異質作業 vs 一般作業

| 特性 | 一般作業 | 異質作業 |
|------|----------|----------|
| 組件數 | 1 | 多個 |
| job_id | 1 個 | 每組件各 1 個 |
| 資源需求 | 統一 | 每組件可不同 |
| 排程 | 獨立 | 所有組件同時 |
| MPI_COMM_WORLD | 單一 | 可合併或分開 |

### --het-group 使用方式

| 選項 | 說明 |
|------|------|
| `--het-group=0` | 在組件 0 上啟動 |
| `--het-group=1` | 在組件 1 上啟動 |
| `--het-group=0,1` | 在組件 0 和 1 上啟動 |
| `--het-group=0-2` | 在組件 0、1、2 上啟動 |
| 不指定 | 預設在組件 0 上啟動 |

---

## Practical Example

### 場景：CPU+GPU 混合計算

某些計算需要一個主節點進行協調（需要大記憶體），以及多個 GPU 節點進行實際計算。

```bash
# 方法 1：命令列提交
$ sbatch -N1 --mem=256G -C haswell : \
         -N4 --gres=gpu:4 -C knl my_hetjob.sh

# 方法 2：腳本內指定
$ cat my_hetjob.sh
#!/bin/bash
#SBATCH -J hetjob_example
#SBATCH -N1 --mem=256G --cpus-per-task=8
#SBATCH hetjob
#SBATCH -N4 --gres=gpu:4 --ntasks-per-node=4

# 在組件 0（主節點）上啟動伺服器
srun --het-group=0 ./coordinator &

# 在組件 1（GPU 節點）上啟動工作者
srun --het-group=1 ./gpu_worker &

wait

$ sbatch my_hetjob.sh
```

### 場景：單一 MPI_COMM_WORLD

將不同資源的組件合併為一個 MPI 程式：

```bash
$ salloc -N1 --mem=256GB -C haswell : \
         -n32 -N4 --ntasks-per-core=1 -C knl bash

# 啟動合併的 MPI 應用程式（單一 MPI_COMM_WORLD）
$ srun server : client
```

### 查詢與管理異質作業

```bash
# 查看異質作業狀態
$ squeue --job=123
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
123+0     debug  bash  user  R 10:00      1 node01
123+1       gpu  bash  user  R 10:00      4 node[02-05]

# 查看特定組件詳情
$ scontrol show job 123+1

# 取消特定組件
$ scancel 123+1

# 取消整個異質作業
$ scancel 123

# 會計查詢
$ sacct -j 123 --format=JobID,JobName,Partition,AllocCPUs,State
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **誤解組件編號**
   ```bash
   # 錯誤：以為組件從 1 開始
   srun --het-group=1 ./app  # 實際上是第二個組件

   # 正確：組件從 0 開始
   srun --het-group=0 ./app  # 第一個組件
   ```

2. **忘記 het-group 選項**
   ```bash
   # 問題：想在 GPU 組件執行，但沒指定 het-group
   salloc -N1 --mem=64G : -N4 --gres=gpu:4 bash
   srun ./gpu_app  # 只在組件 0（無 GPU）執行！

   # 正確：指定 het-group
   srun --het-group=1 ./gpu_app
   ```

3. **重疊分割區導致自我餓死**
   ```bash
   # 問題：兩個分割區共享節點，請求可能永遠無法滿足
   sbatch -p p1 -N5 : -p p2 -N5 job.sh
   # 如果 p1 和 p2 共享節點，可能導致作業無法啟動

   # 解決：確保分割區不重疊或減少請求
   ```

4. **期望 mpirun 支援異質作業**
   ```bash
   # 錯誤：mpirun 不支援異質作業
   mpirun -np 4 : -np 8 ./app  # 不會工作

   # 正確：使用 srun
   srun -n4 : -n8 ./app
   ```

### 實用建議

- **使用腳本指定異質作業**：比命令列更清晰易讀
- **測試小規模**：先用少量節點測試異質作業配置
- **監控所有組件**：使用 `squeue --job=<het_job_id>` 查看所有組件狀態
- **理解環境變數**：每個組件有自己的 `_HET_GROUP_N` 環境變數
- **考慮依賴關係**：所有組件同時排程，確保資源足夠

---

## Quick Reference

### 提交異質作業

```bash
# 命令列方式
sbatch <opts1> : <opts2> : <opts3> script.sh

# 腳本方式
#SBATCH <opts1>
#SBATCH hetjob
#SBATCH <opts2>
```

### 作業 ID 格式

| 格式 | 說明 |
|------|------|
| `123` | 異質作業領導者 ID（所有組件）|
| `123+0` | 第一個組件 |
| `123+1` | 第二個組件 |
| `123+N` | 第 N+1 個組件 |

### 常用指令

| 指令 | 說明 |
|------|------|
| `srun --het-group=N ./app` | 在組件 N 上啟動應用程式 |
| `srun app1 : app2` | 在不同組件啟動不同應用程式 |
| `squeue --job=<het_job_id>` | 查看所有組件 |
| `scancel <het_job_id>` | 取消所有組件 |
| `scancel <het_job_id>+N` | 只取消組件 N |
| `scontrol show job <id>+N` | 查看特定組件詳情 |

### 環境變數

| 變數 | 說明 |
|------|------|
| `SLURM_JOB_ID` | 異質作業領導者 ID |
| `SLURM_HET_SIZE` | 組件總數 |
| `SLURM_JOB_ID_HET_GROUP_N` | 組件 N 的 job_id |
| `SLURM_JOB_NODELIST_HET_GROUP_N` | 組件 N 的節點列表 |
| `SLURM_JOB_NODES_HET_GROUP_N` | 組件 N 的節點數 |
