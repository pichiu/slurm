# Slurm CPU 管理使用者與管理員指南 (CPU Management Guide)

## TL;DR

Slurm 使用四個步驟管理 CPU 資源：(1) 選擇節點、(2) 從節點分配 CPU、(3) 分派任務到節點、(4) 選擇性地將任務綁定到特定 CPU。關鍵設定包括 `SelectType`（select/linear 或 select/cons_tres）和 `SelectTypeParameters`（CR_Core、CR_CPU 等）。使用者可透過 `--ntasks`、`--cpus-per-task`、`--nodes` 和 `--cpu-bind` 等選項控制資源分配。

---

## 翻譯

### 概觀

本指南旨在協助 Slurm 使用者和管理員選擇設定選項並組合命令列來管理作業、步驟和任務對 CPU 資源的使用。文件分為以下章節：

- 概觀
- Slurm 執行的 CPU 管理步驟
- 取得作業/步驟/任務的 CPU 使用資訊
- CPU 管理與 Slurm 計費
- CPU 管理範例

透過使用者指令進行的 CPU 管理受限於 Slurm 管理員選擇的設定參數。不同 CPU 管理選項之間的互動複雜且通常難以預測。可能需要一些實驗才能找到產生預期結果的確切選項組合。

相關文件：
- [Slurm 可消耗資源](cons_tres.html)
- [共享可消耗資源](cons_tres_share.html)
- [多核心/多執行緒架構支援](mc_support.md)
- [平面分佈](dist_plane.html)

---

### Slurm 執行的 CPU 管理步驟

Slurm 使用四個基本步驟來管理作業/步驟的 CPU 資源：

#### 步驟 1：選擇節點

在步驟 1 中，Slurm 選擇一組節點，從這些節點分配 CPU 資源給作業或作業步驟。

| slurm.conf 參數 | 可能值 | 說明 |
|-----------------|--------|------|
| `NodeName` | 節點名稱 + 其他參數 | 定義節點，包括板卡、插槽、核心、執行緒和處理器的數量與佈局 |
| `PartitionName` | 分割區名稱 + 其他參數 | 定義分割區，多個參數會影響節點選擇 |
| `SelectType` | `select/linear` \| `select/cons_tres` | 控制 CPU 資源是以整個節點為單位還是以可消耗資源分配 |
| `SelectTypeParameters` | `CR_CPU` \| `CR_Core` \| `CR_Socket` 等 | 定義可消耗資源類型 |

**主要命令列選項（srun/salloc/sbatch）：**

| 選項 | 說明 |
|------|------|
| `-N, --nodes` | 控制分配給作業的最小/最大節點數 |
| `-n, --ntasks` | 控制要為作業建立的任務數 |
| `-c, --cpus-per-task` | 控制每個任務分配的 CPU 數 |
| `-C, --constraint` | 限制節點選擇到具有指定屬性的節點 |
| `--exclusive` | 防止已分配的節點與其他作業共享 |
| `-w, --nodelist` | 指定要分配給作業的特定節點列表 |
| `-x, --exclude` | 指定要從分配中排除的特定節點列表 |

#### 步驟 2：從選定節點分配 CPU

在步驟 2 中，Slurm 從步驟 1 選定的節點集合中為作業/步驟分配 CPU 資源。

- 若設定 `SelectType=select/linear`：選定節點上的所有資源都會分配給作業/步驟
- 若 `SelectType` 設為 `select/cons_tres`：可從選定節點分配個別的插槽、核心和執行緒作為可消耗資源

**預設分配方法：**
- **跨節點**：區塊分配（block）— 在使用另一個節點之前，先分配一個節點中所有可用的 CPU
- **節點內**：循環分配（cyclic）— 在節點內的插槽之間以輪詢方式分配可用的 CPU

#### 步驟 3：將任務分派到選定節點

在步驟 3 中，Slurm 將任務分派到步驟 1 中為作業/步驟選定的節點。每個任務只會分派到一個節點，但可以有多個任務分派到同一個節點。

| slurm.conf 參數 | 說明 |
|-----------------|------|
| `MaxTasksPerNode` | 控制作業步驟可在單一節點上產生的最大任務數 |

**命令列選項：**

| 選項 | 說明 |
|------|------|
| `-m, --distribution` | 控制任務分派到各選定節點的順序（block、cyclic、arbitrary、plane） |
| `--ntasks-per-node` | 控制每個分配節點的最大任務數 |
| `--ntasks-per-socket` | 控制每個分配插槽的最大任務數 |
| `--ntasks-per-core` | 控制每個分配核心的最大任務數 |

#### 步驟 4：選擇性地將任務分派並綁定到節點內的 CPU

在選擇性的步驟 4 中，Slurm 將每個任務分派並綁定到步驟 3 中任務被分派到的節點上已分配 CPU 的指定子集。此步驟稱為任務親和性（task affinity）或任務/CPU 綁定。

| slurm.conf 參數 | 可能值 | 說明 |
|-----------------|--------|------|
| `TaskPlugin` | `task/none` \| `task/affinity` \| `task/cgroup` | 控制是否啟用此步驟及使用哪個任務外掛程式 |

| cgroup.conf 參數 | 說明 |
|------------------|------|
| `ConstrainCores` | 控制作業是否受限於其分配的 CPU |

**命令列選項：**

| 選項 | 說明 |
|------|------|
| `--cpu-bind` | 控制任務與 CPU 的綁定（僅限 srun） |
| `-m, --distribution` | 第二個指定的分佈（冒號後）控制節點內任務與 CPU 的綁定 |

---

### CPU 管理步驟的補充說明

對於可消耗資源，使用者必須了解 CPU 分配（步驟 2）與任務親和性/綁定（步驟 4）之間的差異：

- **獨占（不共享）分配** CPU 作為可消耗資源會限制可同時使用節點的作業/步驟/任務數量
- **但這並不限制**分派到節點的每個任務可使用的 CPU 集合
- 除非使用某種形式的 CPU/任務綁定，否則分派到節點的所有任務都可以使用節點上的所有 CPU，包括未分配給其作業/步驟的 CPU

因此，建議在設定可消耗資源的同時也設定任務親和性。

---

### 取得作業/步驟/任務的 CPU 使用資訊

| 指令/選項 | 資訊 |
|-----------|------|
| `scontrol show job --details` | 提供為作業選定的節點列表，以及每個節點上分配給作業的 CPU ID |
| `env` | Slurm 環境變數提供與節點和 CPU 使用相關的資訊 |
| `srun --cpu-bind=verbose` | 提供任務親和性用於將任務綁定到 CPU 的 CPU 遮罩列表 |
| `srun -l` | 將任務 ID 作為前綴加到發送到 stdout/stderr 的每行輸出 |
| `cat /proc/<pid>/status \| grep Cpus_allowed_list` | 給定任務的 PID，產生綁定到該任務的 CPU ID 列表 |

**重要的 Slurm 環境變數：**
- `SLURM_JOB_CPUS_PER_NODE`
- `SLURM_CPUS_PER_TASK`
- `SLURM_CPU_BIND`
- `SLURM_DISTRIBUTION`
- `SLURM_JOB_NODELIST`
- `SLURM_TASKS_PER_NODE`
- `SLURM_NTASKS`
- `SLURM_CPUS_ON_NODE`

#### CPU 編號說明

Slurm 已知的邏輯 CPU 數量和佈局在 slurm.conf 的節點定義中描述。這可能與實際硬體上的實體 CPU 佈局不同。因此，Slurm 會產生自己的內部或「抽象」CPU 編號。這些編號可能與 Linux 已知的實體或「機器」CPU 編號不符。

---

### CPU 管理與 Slurm 計費

Slurm 使用者的 CPU 管理受 Slurm 計費施加的限制約束。計費限制可在使用者、群組和叢集層級應用於 CPU 使用。詳情請參閱 sacctmgr 手冊頁。

---

## 說明

### SelectType 選項比較

| SelectType | 說明 | 適用場景 |
|------------|------|----------|
| `select/linear` | 以整個節點為單位分配 | 需要專用節點的大型 HPC 作業 |
| `select/cons_tres` | 以可消耗資源（CPU、記憶體等）分配 | 需要更細緻資源管理的環境 |

### SelectTypeParameters 選項

| 參數 | 說明 |
|------|------|
| `CR_CPU` | 以 CPU（執行緒）為可消耗資源 |
| `CR_Core` | 以核心為可消耗資源 |
| `CR_Socket` | 以插槽為可消耗資源 |
| `CR_CPU_Memory` | CPU 和記憶體都是可消耗資源 |
| `CR_Core_Memory` | 核心和記憶體都是可消耗資源 |

### 分佈方法比較

| 分佈方法 | 跨節點行為 | 節點內行為 |
|----------|------------|------------|
| `block` | 填滿一個節點後再使用下一個 | 填滿一個插槽後再使用下一個 |
| `cyclic` | 輪詢方式分配到各節點 | 輪詢方式分配到各插槽 |
| `plane=<size>` | 以指定大小的區塊輪詢分配 | - |
| `arbitrary` | 依照指定順序 | - |

---

## 實務範例

### 範例 1：整個節點分配

```bash
# slurm.conf
SelectType=select/linear

# 命令列：分配最少 2 個完整節點
srun --nodes=2 ./my_program
```

### 範例 2：簡單的核心可消耗資源分配

```bash
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core

# 分配 6 個 CPU（2 個任務，每任務 3 個 CPU）從單一節點
srun --nodes=1-1 --ntasks=2 --cpus-per-task=3 ./my_program
```

### 範例 3：跨節點平衡分配

```bash
# 分配 9 個 CPU（3 個任務，每任務 3 個 CPU）從 3 個節點各分配 3 個
srun --nodes=3-3 --ntasks=3 --cpus-per-task=3 ./my_program
```

### 範例 4：最小化資源碎片化

```bash
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core,CR_CORE_DEFAULT_DIST_BLOCK

# 分配 12 個 CPU，使用最少的節點和插槽
srun --ntasks=12 ./my_program
```

### 範例 5：循環分派任務到節點

```bash
# 分配 12 個 CPU，循環分派任務
srun --nodes=2-2 --ntasks-per-node=3 --distribution=cyclic \
     --ntasks=6 --cpus-per-task=2 ./my_program
```

### 範例 6：平面分佈任務

```bash
# 以區塊大小 2 輪詢分派任務
srun --nodes=3-3 --distribution=plane=2 \
     --ntasks=8 --cpus-per-task=2 ./my_program
```

### 範例 7：CPU 超額配置

```bash
# 允許 16 個任務只使用 8 個 CPU
srun --overcommit --nodes=1-1 --ntasks=16 ./my_program
```

### 範例 8：作業間資源共享

```bash
# slurm.conf 分割區設定
PartitionName=shared Nodes=n0,n1,n2 OverSubscribe=YES

# 命令列
srun --oversubscribe --nodes=1-1 --ntasks=4 ./my_program
```

### 範例 9：多執行緒節點，每核心只分配一個執行緒

```bash
# 使用 --hint 選項避免使用超執行緒
srun --hint=nomultithread --ntasks=8 ./my_program
```

### 範例 10：任務親和性與核心綁定

```bash
# slurm.conf
TaskPlugin=task/affinity

# 將每個任務綁定到核心
srun --cpu-bind=cores --ntasks=4 ./my_program
```

### 範例 11-13：任務親和性與插槽綁定

```bash
# 將任務綁定到插槽
srun --cpu-bind=sockets --ntasks=2 ./my_program

# 使用遮罩指定精確的 CPU 綁定
srun --cpu-bind=mask_cpu:0x0f,0xf0 --ntasks=2 ./my_program
```

### 範例 14：自訂分配與分佈

```bash
# 結合多個選項進行精細控制
srun --nodes=2-2 --ntasks=4 --cpus-per-task=2 \
     --distribution=cyclic:cyclic --cpu-bind=cores ./my_program
```

### 範例 15：優化多任務多執行緒作業效能

```bash
# 對於使用 OpenMP 的 MPI 程式
export OMP_NUM_THREADS=4
srun --nodes=2 --ntasks=4 --cpus-per-task=4 \
     --distribution=block:block --cpu-bind=cores ./hybrid_program
```

### 範例 16：使用 task cgroup

```bash
# slurm.conf
TaskPlugin=task/cgroup

# cgroup.conf
ConstrainCores=yes

# 命令列（與 task/affinity 類似）
srun --cpu-bind=cores --ntasks=4 ./my_program
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 混淆核心與 CPU | CPU 可能指核心或執行緒，取決於 SelectTypeParameters |
| 忽略任務親和性 | 使用可消耗資源時應同時設定 TaskPlugin |
| 錯誤的 --cpus-per-task | 確保請求的 CPU 數與程式實際使用的執行緒數相符 |
| 過度使用 --exclusive | 會導致資源利用率降低 |

### 效能最佳化建議

1. **MPI 程式**：使用 `--ntasks` 和 `--distribution` 控制任務佈局
2. **OpenMP 程式**：使用 `--cpus-per-task` 和 `OMP_NUM_THREADS`
3. **混合 MPI+OpenMP**：結合 `--ntasks`、`--cpus-per-task` 和 `--cpu-bind`
4. **記憶體密集型**：使用 `CR_Core_Memory` 或 `CR_CPU_Memory`
5. **NUMA 感知**：使用 `--cpu-bind=sockets` 或特定的 CPU 遮罩

### 除錯技巧

```bash
# 查看分配詳情
scontrol show job <jobid> --details

# 查看實際的 CPU 綁定
srun --cpu-bind=verbose ./my_program

# 在作業腳本中檢查
cat /proc/self/status | grep Cpus_allowed_list

# 使用環境變數
echo $SLURM_JOB_CPUS_PER_NODE
echo $SLURM_CPUS_ON_NODE
```

---

## 快速參考

### CPU 管理四步驟

| 步驟 | 執行者 | 主要控制選項 |
|------|--------|--------------|
| 1. 選擇節點 | slurmctld + select 外掛程式 | `-N`, `-C`, `--nodelist` |
| 2. 分配 CPU | slurmctld + select 外掛程式 | `--cpus-per-task`, `--distribution` |
| 3. 分派任務 | slurmctld | `-n`, `--ntasks-per-*`, `--distribution` |
| 4. 綁定 CPU | slurmd + task 外掛程式 | `--cpu-bind`, `--distribution` |

### 常用選項組合

| 場景 | 建議選項 |
|------|----------|
| 單節點多核心 | `--nodes=1 --ntasks=1 --cpus-per-task=N` |
| 多節點 MPI | `--nodes=N --ntasks=M --distribution=cyclic` |
| 混合 MPI+OpenMP | `--ntasks=N --cpus-per-task=T --cpu-bind=cores` |
| GPU 程式 | `--gpus=N --cpus-per-gpu=C` |
| 專用節點 | `--exclusive --nodes=N` |

### SelectTypeParameters 快速參考

| 參數 | 資源單位 | 適用情境 |
|------|----------|----------|
| CR_CPU | CPU（執行緒） | 超執行緒系統 |
| CR_Core | 核心 | 一般多核心系統（推薦） |
| CR_Socket | 插槽 | 需要整個插槽的應用 |
| CR_*_Memory | + 記憶體 | 記憶體密集型工作負載 |
