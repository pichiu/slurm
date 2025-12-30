# Slurm 核心專用化 (Core Specialization)

## TL;DR

核心專用化用於隔離系統開銷（系統中斷等）到指定核心，減少應用程式的上下文切換以提升完成時間。使用 `--core-spec` 選項指定保留核心數，需設定 `AllowSpecResourcesUsage=yes`。需要 `SelectType=cons_tres` 和 `task/cgroup`。可透過 `CoreSpecCount` 或 `CPUSpecList` 在節點設定中指定專用核心。

---

## 翻譯

### 概觀

核心專用化是一項功能，旨在將系統開銷（系統中斷等）隔離到運算節點上的指定核心。這可以減少應用程式的上下文切換以改善完成時間。作業程序將無法直接使用專用化的核心。

---

### 命令選項

所有作業分配命令（`salloc`、`sbatch` 和 `srun`）都接受 `-S` 或 `--core-spec` 選項，帶有核心數值參數。

**使用範例**：
```bash
sbatch -S 1 script.sh        # 保留 1 個核心
salloc --core-spec=2         # 保留 2 個核心
```

**注意事項**：
- 若 slurm.conf 中未啟用 `AllowSpecResourcesUsage`，`--core-spec` 選項將被忽略
- 可使用 `scontrol`、`sview` 或 `squeue` 命令查看每個作業的核心專用化數量
- 作業步驟（job step）的核心專用化數量規格會被忽略
- 使用 `squeue` 的 "%X" 格式選項查看數量（預設輸出格式中不會報告）
- `scontrol` 和 `sview` 命令也可修改等待中作業的數量

#### 隱式獨佔模式

明確設定作業的專用核心值會**隱式設定** `--exclusive` 選項，為作業保留整個節點。

**影響**：
- 作業將被收取節點上所有非專用 CPU 的費用
- `scontrol`、`sview` 和 `squeue` 報告的 NumCPUs 值將反映所有已分配節點上的所有非專用 CPU
- 作業的計費也是如此

#### 核心存取規則

由於隱式 `--exclusive`，如果請求的專用核心/執行緒數量**低於**已分配節點的 `CoreSpecCount` 或 `CpuSpecList` 中的核心數量，則步驟將可以存取所有非專用核心以及為此作業釋放的專用核心。

**範例**：

假設節點設定：
- `AllowSpecResourcesUsage=yes`
- `CoreSpecCount=2`
- 總共 16 個核心

如果作業指定 `--core-spec=1`：
- 隱式 `--exclusive` 導致節點獨佔分配
- 15 個核心可供作業使用
- 1 個核心保留給系統使用

#### 計費中的 CPU 計數

在 `sacct` 中：
- 步驟的已分配 CPU 將包含其可存取的專用核心或執行緒
- 作業的已分配 CPU 計數**從不**包含專用核心或執行緒，以確保利用率報告準確

**範例設定與輸出**：

```
# slurm.conf
AllowSpecResourcesUsage=yes
Nodename=n0 Port=10100 CoresPerSocket=16 ThreadsPerCore=1 CpuSpecList=0-1
```

```bash
# 提交作業請求 core spec 計數為 1（釋放核心 1 供作業使用）
$ salloc --core-spec=1
salloc: Granted job allocation 4152

$ srun bash -c 'cat /proc/self/status |grep Cpus_'
Cpus_allowed:        fffe
Cpus_allowed_list:   1-15

# 注意作業 CPU 計數 vs 步驟 CPU 計數
$ sacct -j 4152 -ojobid%20,alloccpus
               JobID  AllocCPUS
-------------------- ----------
                4152         14
    4152.interactive         15
              4152.0         15
```

---

### 核心選擇

專用化要使用的特定資源可透過 slurm.conf 檔案中與每個節點相關的 `CPUSpecList` 設定參數來識別。

#### 自動選擇演算法

如果設定了 `CoreSpecCount` 但沒有 `CPUSpecList`，專用化核心將遵循以下演算法選擇：

**預設（從最後核心開始）**：
1. 首先選擇最高編號 socket 上的最高編號核心
2. 後續選擇較低編號 socket 上的最高編號核心
3. 如果需要更多核心，則來自每個 socket 上次高編號的核心

**範例**：兩個 socket，每個有四個核心的節點

選擇順序：
| 順序 | Socket | Core |
|------|--------|------|
| 1 | 1 | 3 |
| 2 | 0 | 3 |
| 3 | 1 | 2 |
| 4 | 0 | 2 |
| 5 | 1 | 1 |
| 6 | 0 | 1 |
| 7 | 1 | 0 |
| 8 | 0 | 0 |

#### 從第一個核心開始選擇

設定 `SchedulerParameters=spec_cores_first` 可從第一個核心開始選擇：
- 首先選擇最低編號 socket 上的最低編號核心
- 後續選擇較高編號 socket 上的最低編號核心
- 如果需要更多核心，則來自每個 socket 上次低編號的核心

**注意**：核心專用化保留可能影響某些作業分配請求選項的使用，特別是 `--cores-per-socket`。

---

### 系統設定

#### 必要條件

| 設定 | 說明 |
|------|------|
| `SelectType=cons_tres` | 必須使用 cons_tres |
| `TaskPlugin=task/cgroup` | 必須包含 task/cgroup |
| `AllowSpecResourcesUsage=yes` | 允許使用者控制專用核心數 |

#### 節點設定選項

| 選項 | 說明 |
|------|------|
| `CoreSpecCount` | 保留的核心數量 |
| `CPUSpecList` | 保留的特定 CPU 清單 |
| `MemSpecLimit` | 保留的記憶體量 |

這些資源將使用 Linux cgroups 保留。

#### slurmd 約束

- 運算節點守護程式 slurmd 將被約束到保留的資源
- 除非指定 `TaskPluginParam=SlurmdOffSpec`
- 如果使用 cgroup/v1，slurmstepd 程序也將被約束到保留的資源

#### 其他注意事項

- 在其他設定上，作業的核心專用化選項將被靜默清除
- 每個運算節點的核心數量必須設定，或 CPU 數量必須設定為節點的核心數量
- 如果未設定核心數量且 CPU 值設定為超執行緒數量，則將保留超執行緒而非核心供系統使用

---

## 說明

### 核心專用化的用途

| 用途 | 說明 |
|------|------|
| 隔離系統開銷 | 將系統中斷等開銷隔離到專用核心 |
| 減少上下文切換 | 應用程式不會被系統活動干擾 |
| 提升 HPC 效能 | 對於延遲敏感的應用程式特別有效 |
| 改善可預測性 | 作業執行時間更一致 |

### 核心分配概念圖

```
節點（2 socket × 4 core = 8 核心）
┌─────────────────────────────────┐
│  Socket 0        Socket 1       │
│  ┌───┬───┬───┬───┐  ┌───┬───┬───┬───┐  │
│  │ 0 │ 1 │ 2 │ 3 │  │ 0 │ 1 │ 2 │ 3 │  │
│  └───┴───┴───┴───┘  └───┴───┴───┴───┘  │
└─────────────────────────────────┘

CoreSpecCount=2 時：
- 專用核心：Socket1-Core3, Socket0-Core3
- 作業可用：其餘 6 個核心
```

---

## 實務範例

### 基本設定

```
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
TaskPlugin=task/cgroup

AllowSpecResourcesUsage=yes
```

### 使用 CoreSpecCount

```
# slurm.conf
# 每個節點保留 2 個核心給系統
NodeName=node[01-10] CPUs=16 CoreSpecCount=2
```

### 使用 CPUSpecList

```
# slurm.conf
# 明確指定 CPU 0 和 1 為專用
NodeName=node[01-10] CPUs=16 CpuSpecList=0-1
```

### 保留記憶體

```
# slurm.conf
# 保留 2 個核心和 1GB 記憶體
NodeName=node[01-10] CPUs=16 CoreSpecCount=2 MemSpecLimit=1024
```

### 使用者覆蓋

```bash
# 使用者請求只保留 1 個核心（低於系統設定的 2 個）
salloc --core-spec=1
```

### 查看專用化資訊

```bash
# 查看作業的 core spec
squeue -o "%.10i %.9P %.8j %.8u %.2t %.4X"

# 詳細檢視
scontrol show job <jobid> | grep CoreSpec

# 查看節點設定
scontrol show node <nodename> | grep -E "CoreSpec|CpuSpecList"
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 AllowSpecResourcesUsage | 設定 `AllowSpecResourcesUsage=yes` |
| 使用錯誤的 SelectType | 必須使用 `select/cons_tres` |
| 未設定 task/cgroup | TaskPlugin 必須包含 `task/cgroup` |
| CPU 數量設定為超執行緒數 | 設定為核心數量以保留核心而非超執行緒 |
| 忽略隱式 exclusive | 理解 core-spec 會觸發獨佔模式 |

### 使用建議

1. **選擇適當的專用核心數**：
   - 一般：1-2 個核心
   - 高系統開銷環境：2-4 個核心
   - 根據應用程式測試調整

2. **使用 CPUSpecList 精確控制**：
   - 可選擇特定的 NUMA 節點上的核心
   - 避免影響應用程式的 NUMA 布局

3. **監控效果**：
   ```bash
   # 比較有無 core spec 的執行時間
   time srun --core-spec=0 ./app
   time srun --core-spec=2 ./app
   ```

4. **與 cgroup 記憶體限制結合**：
   - 使用 MemSpecLimit 確保系統有足夠記憶體
   - 避免 OOM killer 終止系統程序

---

## 快速參考

### slurm.conf 設定

```
# 必要設定
SelectType=select/cons_tres
TaskPlugin=task/cgroup
AllowSpecResourcesUsage=yes

# 節點設定選項
NodeName=... CoreSpecCount=N        # 保留 N 個核心
NodeName=... CpuSpecList=0-1        # 保留特定 CPU
NodeName=... MemSpecLimit=1024      # 保留記憶體 (MB)

# 可選：從第一個核心開始選擇
SchedulerParameters=spec_cores_first
```

### 命令選項

| 命令 | 說明 |
|------|------|
| `sbatch --core-spec=N` | 保留 N 個核心 |
| `salloc -S N` | 同上（簡短形式）|
| `squeue -o "%X"` | 顯示 core spec 值 |
| `scontrol show job` | 顯示詳細作業資訊 |

### 相關文件

- [可消耗資源](cons_tres.md) - 資源選擇設定
- [Cgroups](cgroups.md) - cgroup 設定
- [資源綁定](../users/resource_binding.md) - 資源綁定設定
