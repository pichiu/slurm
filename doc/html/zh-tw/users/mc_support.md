# Slurm 多核心/多執行緒架構支援 (Multi-core/Multi-thread Support)

## TL;DR

Slurm 提供完整的多核心和多執行緒架構支援。關鍵概念包括：插槽 (Socket)、核心 (Core) 和執行緒 (Thread)。使用者可透過 `--cpu-bind`（低階綁定）或 `-B S:C:T`（高階自動遮罩生成）控制 CPU 親和性。任務分佈選項 (`-m/--distribution`) 支援 block、cyclic 和 plane 等模式。需在 slurm.conf 中啟用 `TaskPlugin=task/affinity` 或 `task/cgroup` 才能使用 CPU 綁定功能。

---

## 翻譯

### 目錄

- 定義
- srun 旗標概觀
- 高階 srun 旗標的設計動機
- sinfo/squeue/scontrol 的擴充
- slurm.conf 中的設定

---

### 定義

| 術語 | 說明 |
|------|------|
| **BaseBoard** | 也稱為主機板 |
| **LDom** | 區域網域（Locality domain）或 NUMA 網域。可能等同於 BaseBoard 或 Socket |
| **Socket/Core/Thread** | 插槽/核心/執行緒 - 見下圖說明 |
| **CPU** | 根據系統設定，可以是核心或執行緒 |
| **Affinity（親和性）** | 綁定到特定邏輯處理器的狀態 |
| **Affinity Mask（親和性遮罩）** | 位元遮罩，索引對應到邏輯處理器。最低有效位元對應系統上第一個邏輯處理器編號，最高有效位元對應最後一個邏輯處理器編號。'1' 表示程序可在該邏輯處理器上執行 |
| **Fat Masks** | 設置超過 1 個位元的親和性遮罩，允許程序在多個邏輯處理器上執行 |

**Socket、Core、Thread 的層次關係：**
```
節點 (Node)
├── 插槽 0 (Socket 0)
│   ├── 核心 0 (Core 0)
│   │   ├── 執行緒 0 (Thread 0) = CPU
│   │   └── 執行緒 1 (Thread 1) = CPU
│   └── 核心 1 (Core 1)
│       ├── 執行緒 0
│       └── 執行緒 1
└── 插槽 1 (Socket 1)
    └── ...
```

---

### srun 旗標概觀

#### 低階選項（明確綁定）

| 選項 | 說明 |
|------|------|
| `--cpu-bind=...` | 明確的程序親和性綁定和控制選項 |

#### 高階選項（自動遮罩生成）

| 選項 | 說明 |
|------|------|
| `--sockets-per-node=S` | 每個節點專用於作業的插槽數（最小值） |
| `--cores-per-socket=C` | 每個插槽專用於作業的核心數（最小值） |
| `--threads-per-core=T` | 每個核心專用於作業的執行緒數（最小值） |
| `-B S[:C[:T]]` | 上述三個選項的組合捷徑 |

#### 任務分佈選項

| 選項 | 說明 |
|------|------|
| `-m / --distribution` | 分佈方式：arbitrary \| block \| cyclic \| plane=x \| [block\|cyclic]:[block\|cyclic\|fcyclic] |

#### 記憶體作為可消耗資源

| 選項 | 說明 |
|------|------|
| `--mem=mem` | 作業每個節點所需的實體記憶體量 |
| `--mem-per-cpu=mem` | 每個分配的 CPU 所需的實體記憶體量 |

#### 任務呼叫控制

| 選項 | 說明 |
|------|------|
| `--cpus-per-task=CPUs` | 每個任務所需的 CPU 數 |
| `--ntasks-per-node=ntasks` | 每個節點要呼叫的任務數 |
| `--ntasks-per-socket=ntasks` | 每個插槽要呼叫的任務數 |
| `--ntasks-per-core=ntasks` | 每個核心要呼叫的任務數 |
| `--overcommit` | 允許每個 CPU 執行多個任務 |

#### 應用程式提示

| 選項 | 說明 |
|------|------|
| `--hint=compute_bound` | 使用每個插槽中的所有核心 |
| `--hint=memory_bound` | 每個插槽只使用一個核心 |
| `--hint=[no]multithread` | [不要] 使用核心內多執行緒的額外執行緒 |

#### 系統保留資源

| 選項 | 說明 |
|------|------|
| `--core-spec=cores` | 為系統使用保留的核心數 |
| `--thread-spec=threads` | 為系統使用保留的執行緒數 |

**重要說明**：這些旗標只有在程序對特定 CPU 和（可選的）記憶體有某種親和性時才有意義。任務親和性使用 slurm.conf 中的 TaskPlugin 參數設定。除了 "task/none" 之外的任何選項都會將任務綁定到 CPU。

---

### 低階 --cpu-bind - 明確綁定介面

```
--cpu-bind=        將任務綁定到 CPU
    q[uiet]         在任務執行前靜默綁定（預設）
    v[erbose]       在任務執行前詳細報告綁定
    no[ne]          不將任務綁定到 CPU（預設）
    map_cpu:<list>  為每個任務指定 CPU ID 綁定
                    其中 <list> 是 <cpuid1>,<cpuid2>,...<cpuidN>
    mask_cpu:<list> 為每個任務指定 CPU ID 綁定遮罩
                    其中 <list> 是 <mask1>,<mask2>,...<maskN>
    rank_ldom       依排名將任務綁定到 NUMA 區域網域中的 CPU
    map_ldom:<list> 為每個任務指定 NUMA 區域網域 ID
    mask_ldom:<list> 為每個任務指定 NUMA 區域網域 ID 遮罩
    ldoms           自動生成的遮罩綁定到 NUMA 區域網域
    sockets         自動生成的遮罩綁定到插槽
    cores           自動生成的遮罩綁定到核心
    threads         自動生成的遮罩綁定到執行緒
    help            顯示此說明訊息
```

**範例：**
```bash
srun -n 8 -N 4 --cpu-bind=mask_cpu:0x1,0x4 a.out
srun -n 8 -N 4 --cpu-bind=mask_cpu:0x3,0xD a.out
```

---

### 高階 -B S[:C[:T]] - 自動遮罩生成介面

使用者可以請求特定數量的節點、插槽、核心和執行緒：

```
-B, --extra-node-info=S[:C[:T]]            展開為：
  --sockets-per-node=S   每個節點分配的插槽數
  --cores-per-socket=C   每個插槽分配的核心數
  --threads-per-core=T   每個核心分配的執行緒數
                每個欄位可以是 'min' 或萬用字元 '*'

     請求的總 CPU 數 = (Nodes) x (S x C x T)
```

**範例：**
```bash
srun -n 8 -N 4 -B 2:1 a.out
srun -n 8 -N 4 -B 2   a.out
srun -n 16 -N 4 -B 2:2:1 a.out
srun -n 16 -N 4 --sockets-per-node=2 --cores-per-socket=2 --threads-per-core=1 a.out
srun -n 16 -N 2-4 -B '1:*:1' a.out
```

**注意事項：**
- 在命令列加上 `--cpu-bind=no` 會使程序不綁定到邏輯處理器
- 加上 `--cpu-bind=verbose` 會讓每個任務報告使用的親和性遮罩
- 使用 `-B` 時預設啟用綁定

---

### 任務分佈選項擴充

`-m / --distribution` 選項已擴充為也描述最低層級邏輯處理器內的分佈：

```
arbitrary | block | cyclic | plane=x | [block|cyclic]:[block|cyclic|fcyclic]
```

- **plane 分佈**（plane=x）：產生區塊大小等於 x 的 block:cyclic 分佈
- **二維分佈**：第二個分佈（冒號後）允許使用者指定節點內程序的分佈方法
  - `cyclic`：將所有 CPU 作為一組分配（如果可能在同一插槽內）
  - `fcyclic`：以循環方式跨插槽分配每個 CPU

**範例：**
```bash
srun -n 16 -N 4 -B '2:*:1' -m block:cyclic --cpu-bind=socket a.out
srun -n 16 -N 4 -B '2:*:1' -m plane=2 --cpu-bind=core a.out
```

預設分佈等同於 `-m block:cyclic` 配合 `--cpu-bind=thread`。

---

### 記憶體作為可消耗資源

```
--mem=MB      作業每個節點所需的最大實體記憶體量
```

此旗標允許排程器在特定節點上共同分配作業，只要它們的總記憶體需求不超過節點上的總記憶體量。

**設定範例：**
```
# 僅使用記憶體作為可消耗資源
SelectType=select/linear
SelectTypeParameters=CR_Memory

# 或與 CPU 結合
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
```

---

### 任務呼叫與邏輯處理器的關係

`--ntasks-per-{node,socket,core}=ntasks` 旗標允許使用者請求在每個節點、插槽或核心上呼叫不超過 ntasks 個任務。

**範例（雙插槽雙核心叢集）：**
```bash
% srun -n 4 hostname
hydra12
hydra12
hydra12
hydra12

% srun -n 4 --ntasks-per-node=1 hostname
hydra12
hydra13
hydra14
hydra15

% srun -n 4 --ntasks-per-node=2 hostname
hydra12
hydra12
hydra13
hydra13

% srun -n 4 --ntasks-per-socket=1 hostname
hydra12
hydra12
hydra13
hydra13

% srun -n 4 --ntasks-per-core=1 hostname
hydra12
hydra12
hydra12
hydra12
```

---

### 應用程式提示

不同的應用程式有不同的資源需求：

- **計算密集型**：使用多核心系統中的所有核心
- **記憶體密集型**：每個插槽只使用一個核心可獲得最高的每核心記憶體頻寬
- **通訊密集型**：使用核心內多執行緒（如超執行緒）可能提高效能

```
--hint=             根據應用程式提示綁定任務
    compute_bound   使用每個插槽中的所有核心
    memory_bound    每個插槽只使用一個核心
    [no]multithread [不要] 使用核心內多執行緒的額外執行緒
    help            顯示此說明訊息
```

**範例：**
```bash
# 計算密集型：使用同一節點的所有核心
% srun -n 4 --hint=compute_bound --cpu-bind=verbose sleep 1
cpu-bind=MASK - hydra12, task  0  0 [15425]: mask 0x1 set
cpu-bind=MASK - hydra12, task  1  1 [15426]: mask 0x4 set
cpu-bind=MASK - hydra12, task  2  2 [15427]: mask 0x2 set
cpu-bind=MASK - hydra12, task  3  3 [15428]: mask 0x8 set

# 記憶體密集型：分散到多個節點
% srun -n 4 --hint=memory_bound --cpu-bind=verbose sleep 1
cpu-bind=MASK - hydra12, task  0  0 [15550]: mask 0x1 set
cpu-bind=MASK - hydra12, task  1  1 [15551]: mask 0x4 set
cpu-bind=MASK - hydra13, task  2  0 [14974]: mask 0x1 set
cpu-bind=MASK - hydra13, task  3  1 [14975]: mask 0x4 set
```

---

### 高階 vs 低階旗標的設計動機

使用高階旗標（如 `-B`）比 `--cpu-bind` 更容易：

1. **自動遮罩生成**：使用高階旗標時自動產生親和性遮罩
2. **簡潔性**：高階旗標的組合比 `--cpu-bind` 更短且更容易使用
3. **可攜性**：使用高階旗標時，Slurm 會為每個請求的節點正確生成適當的遮罩

**高階旗標範例：**
```bash
$ srun -n 32 -N 4 -B 2:2:1 --distribution=block:cyclic a.out
# 或
$ srun -n 32 -N 4 -B 2:2:1 a.out  # block:cyclic 是預設值
```

**低階旗標範例（需要知道核心編號）：**
```bash
# 區塊編號系統
$ srun -n 32 -N 4 --cpu-bind=mask_cpu:1,4,10,40,2,8,20,80 a.out
# 或
$ srun -n 32 -N 4 --cpu-bind=map_cpu:0,2,4,6,1,3,5,7 a.out
```

**重要結論**：使用 `--cpu-bind` 旗標並不簡單，它假設使用者是專家。

---

### sinfo/squeue/scontrol 擴充

#### sinfo

長版本（`-l`）的節點列表（`-N`）已擴充顯示每個節點的插槽、核心和執行緒：

```bash
% sinfo -lN
NODELIST     NODES PARTITION STATE CPUS S:C:T MEMORY TMP_DISK WEIGHT FEATURES REASON
hydra[12-14]     3    parts*  idle    8 2:4:1   2007    41447      1   (null)   none
hydra15          1    parts*  idle   64 8:4:2   2007    41447      1   (null)   none
```

**格式識別碼：**
| 識別碼 | 說明 |
|--------|------|
| `%X` | 每個節點的插槽數 |
| `%Y` | 每個插槽的核心數 |
| `%Z` | 每個核心的執行緒數 |
| `%z` | 擴充處理器資訊：S:C:T |

#### squeue

**格式識別碼：**
| 識別碼 | 說明 |
|--------|------|
| `%m` | 作業請求的記憶體大小（MB） |
| `%H` | 請求的每節點插槽數 |
| `%I` | 請求的每插槽核心數 |
| `%J` | 請求的每核心執行緒數 |
| `%z` | 擴充處理器資訊：S:C:T |

```bash
% squeue -o '%.5i %.2t %.4M %.5D %7H %6I %7J %6z %R'
JOBID ST TIME NODES SOCKETS CORES THREADS S:C:T NODELIST(REASON)
   17 PD 0:00     1 2       2     1       2:2:1 (Resources)
   13  R 1:27     1 2       2     1       2:2:1 hydra12
```

#### scontrol

```bash
# 調整作業設定
scontrol update JobID=17 ReqThreads=2
scontrol update JobID=18 ReqCores=4
scontrol update JobID=19 ReqSockets=8

# 顯示作業詳情
% scontrol show job 20
...
AllocCPUs=1,2,1
NumCPUs=4 ReqS:C:T=2:1:*
...
```

---

### slurm.conf 設定

#### 啟用任務親和性

```
TaskPlugin=task/affinity          # 啟用任務親和性
```

或使用 cgroup：

```
TaskPlugin=task/cgroup            # 使用 Linux cgroup 綁定任務
```

#### 宣告節點硬體設定

```
NodeName=dualcore[01-16] CoresPerSocket=2 ThreadsPerCore=1
```

---

## 實務範例

### 範例 1：基本多核心作業

```bash
# 使用 4 個節點，每節點 2 個插槽，每插槽 2 個核心
srun -n 16 -N 4 -B 2:2:1 ./my_program
```

### 範例 2：記憶體密集型作業

```bash
# 每個插槽只使用一個核心以最大化記憶體頻寬
srun -n 4 --hint=memory_bound ./memory_intensive_program
```

### 範例 3：計算密集型作業

```bash
# 使用所有可用核心
srun -n 8 --hint=compute_bound ./compute_intensive_program
```

### 範例 4：混合 MPI+OpenMP 作業

```bash
export OMP_NUM_THREADS=4
srun -n 4 --cpus-per-task=4 --cpu-bind=cores ./hybrid_program
```

### 範例 5：避免超執行緒

```bash
# 不使用超執行緒
srun -n 8 --hint=nomultithread ./single_thread_program
```

### 範例 6：查看綁定詳情

```bash
srun -n 4 --cpu-bind=verbose,cores ./my_program
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未啟用 TaskPlugin | 在 slurm.conf 中設定 `TaskPlugin=task/affinity` 或 `task/cgroup` |
| 混淆 CPU 和核心 | 了解您系統中 CPU 的定義（核心或執行緒） |
| 使用錯誤的核心編號 | 檢查 `/proc/cpuinfo` 確認系統的核心編號方式 |
| 過度指定限制 | 選項之間可能衝突，從簡單開始 |

### 效能最佳化建議

1. **計算密集型應用**：使用 `--hint=compute_bound`
2. **記憶體密集型應用**：使用 `--hint=memory_bound`
3. **NUMA 感知**：使用 `--cpu-bind=ldoms` 或 `rank_ldom`
4. **避免超執行緒干擾**：使用 `--hint=nomultithread`

### 除錯技巧

```bash
# 查看系統拓撲
sinfo -lN

# 查看作業的 CPU 分配
scontrol show job <jobid>

# 執行時查看綁定
srun --cpu-bind=verbose ./program

# 檢查實際綁定
cat /proc/self/status | grep Cpus_allowed_list
```

---

## 快速參考

### 常用選項組合

| 場景 | 建議選項 |
|------|----------|
| 單節點多核心 | `-N 1 -n T --cpus-per-task=1` |
| 每插槽一個任務 | `--ntasks-per-socket=1` |
| 每節點一個任務 | `--ntasks-per-node=1` |
| 計算密集型 | `--hint=compute_bound` |
| 記憶體密集型 | `--hint=memory_bound` |
| 避免超執行緒 | `--hint=nomultithread` |
| 核心綁定 | `--cpu-bind=cores` |
| 插槽綁定 | `--cpu-bind=sockets` |

### -B 選項快速參考

```
-B S:C:T
   S = 每節點插槽數
   C = 每插槽核心數
   T = 每核心執行緒數

   * = 使用所有可用
   min = 使用最小值
```

### TaskPlugin 選項

| 選項 | 說明 |
|------|------|
| `task/none` | 無任務啟動動作（預設） |
| `task/affinity` | CPU 親和性支援 |
| `task/cgroup` | 使用 Linux cgroup 綁定任務到資源 |
