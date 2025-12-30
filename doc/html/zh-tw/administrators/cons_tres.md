# Slurm 可消耗資源 (Consumable Resources)

## TL;DR

Slurm 預設以獨佔模式分配整個節點給作業，即使節點資源未完全利用。`select/cons_tres` 外掛程式可追蹤 CPU、記憶體、GPU 等可消耗資源，允許多個作業共享同一節點。設定 `SelectType=select/cons_tres` 和 `SelectTypeParameters=CR_Core_Memory` 等參數。用戶可使用 `--exclusive` 請求獨佔節點。支援 GPU 相關選項如 `--gpus`、`--gpus-per-node` 等。

---

## 翻譯

### 概觀

Slurm 使用預設節點分配外掛程式時，以獨佔模式將節點分配給作業。這意味著即使節點內的所有資源未被給定作業利用，其他作業也無法存取這些資源。

節點具有處理器、記憶體、交換空間、本地磁碟等資源，作業則消耗這些資源。Slurm 的獨佔使用預設策略可能導致叢集及其節點資源的低效率利用。

Slurm 的 **cons_tres** 外掛程式可用於以更細緻的方式管理資源。

---

### 使用可消耗可追蹤資源外掛程式：select/cons_tres

可消耗可追蹤資源 (cons_tres) 外掛程式可追蹤多種資源：

| 參數 | 說明 |
|------|------|
| **CR_CPU** | CPU 作為可消耗資源 |
| **CR_Board** | 基板作為可消耗資源 |
| **CR_Socket** | 插槽作為可消耗資源 |
| **CR_Core** | 核心作為可消耗資源 |
| **CR_Socket_Memory** | 插槽和記憶體作為可消耗資源 |
| **CR_Core_Memory** | 核心和記憶體作為可消耗資源 |
| **CR_CPU_Memory** | CPU 和記憶體作為可消耗資源 |

#### CR_CPU 說明

- 沒有插槽、核心或執行緒的概念
- 在多核心系統上，CPU 就是核心
- 在多核心/超執行緒系統上，CPU 就是執行緒
- 在單核心系統上，CPU 就是 CPU

**注意**：所有 CR_* 參數假設 `OverSubscribe=No` 或 `OverSubscribe=Force`。

---

### GPU 功能

cons_tres 外掛程式提供 GPU 專用功能。

#### 設定參數

| 參數 | 說明 |
|------|------|
| **DefCpuPerGPU** | 每個 GPU 分配的預設 CPU 數 |
| **DefMemPerGPU** | 每個 GPU 分配的預設記憶體量 |

#### 作業提交選項

| 選項 | 說明 |
|------|------|
| `--cpus-per-gpu=` | 每個 GPU 的 CPU 數 |
| `--gpus=` | 整個作業分配的 GPU 數 |
| `--gpu-bind=` | 將任務綁定到特定 GPU |
| `--gpu-freq=` | 請求特定 GPU/記憶體頻率 |
| `--gpus-per-node=` | 每個節點的 GPU 數 |
| `--gpus-per-socket=` | 每個插槽的 GPU 數 |
| `--gpus-per-task=` | 每個任務的 GPU 數 |
| `--mem-per-gpu=` | 每個 GPU 的記憶體量 |

---

### 記憶體設定

當記憶體是可消耗資源時，必須在 slurm.conf 中設定 **RealMemory** 參數來定義節點的實際記憶體量。

#### 記憶體請求選項

| 選項 | 說明 |
|------|------|
| `--mem=MB` | 每個節點所需的最大實際記憶體 |
| `--mem-per-cpu=MB` | 每個分配的 CPU 所需的最大實際記憶體 |

**重要**：指定足夠的記憶體很重要，因為 Slurm 不會允許應用程式使用超過請求的實際記憶體量。`--mem` 的預設值繼承自 **DefMemPerNode**。

---

### 啟用外掛程式

```
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
```

---

### 一般說明

- **select/linear** 使用基於連續節點數的最佳擬合演算法
- **select/cons_tres** 在叢集範圍內啟用或停用
- 使用 cons_tres 時，當資源允許時，作業可以在節點上共同排程
- 可追蹤通用資源（如 GPU）

#### 獨佔模式

即使啟用可消耗資源，使用者也可以使用 `--exclusive` srun 選項請求獨佔節點。

#### 過度訂閱

srun 的 `-s` 或 `--oversubscribe` 與可消耗資源環境不相容。由於此環境的節點預設是共享的，使用 `--exclusive` 允許使用者獲得專用節點。

**注意**：`--oversubscribe` 和 `--exclusive` 選項在作業提交時是互斥的。

---

## 說明

### 獨佔模式 vs 可消耗資源

| 特性 | 獨佔模式 (select/linear) | 可消耗資源 (select/cons_tres) |
|------|--------------------------|------------------------------|
| 節點分配 | 整個節點 | 資源級別 |
| 資源共享 | 否 | 是 |
| 適用場景 | 大型並行作業 | 串列或適度並行作業 |
| 效率 | 可能較低 | 較高 |
| 設定複雜度 | 較低 | 較高 |

### 資源消耗概念

```
節點資源
├── CPU (CR_CPU / CR_Core)
│   ├── 可被多個作業共享
│   └── 追蹤已分配/可用數量
├── 記憶體 (CR_*_Memory)
│   ├── 需要設定 RealMemory
│   └── 使用 --mem 或 --mem-per-cpu 請求
└── GPU (GRES)
    ├── 使用 --gpus-* 選項請求
    └── 可設定預設 CPU/記憶體比率
```

---

## 實務範例

### 基本設定

**slurm.conf：**
```
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# GPU 預設值
DefCpuPerGPU=2
DefMemPerGPU=4096
```

### CR_Socket_Memory 範例

```bash
# 節點設定：5 節點，每節點 4 CPU (2 插槽 × 2 核心)，2007 MB 記憶體
sinfo -lNe
NODELIST     NODES PARTITION  STATE  CPUS  S:C:T MEMORY
hydra[12-16]     5 allNodes*  ...       4  2:2:1   2007

# 執行第一個作業：使用 5 節點，每節點 1 任務，每節點 1000MB 記憶體
srun -N 5 -n 5 --mem=1000 sleep 100 &

# 嘗試第二個作業：請求 2000MB 記憶體 - 會排隊等待
srun -n 1 -w hydra12 --mem=2000 sleep 100 &

squeue
JOBID PARTITION   NAME   USER ST  TIME  NODES NODELIST(REASON)
 1890  allNodes  sleep sballe PD  0:00      1 (Resources)
 1889  allNodes  sleep sballe  R  0:08      5 hydra[12-16]
```

### CR_CPU_Memory 範例

```bash
# 第一個作業：使用每節點 1 CPU，1000MB 記憶體
srun -N 5 -n 5 --mem=1000 sleep 100 &

# 第二個作業：可以同時執行（不同 CPU，少量記憶體）
srun -N 5 -n 5 --mem=10 sleep 100 &

# 第三個作業：排隊等待（記憶體不足）
srun -N 5 -n 5 --mem=1000 sleep 100 &

squeue
JOBID PARTITION   NAME   USER ST  TIME  NODES NODELIST(REASON)
 1835  allNodes  sleep sballe PD  0:00      5 (Resources)
 1833  allNodes  sleep sballe  R  0:10      5 hydra[12-16]
 1834  allNodes  sleep sballe  R  0:07      5 hydra[12-16]
```

### 節點分配比較

**叢集設定**：
- linux01: 2 CPU
- linux02: 2 CPU
- linux03: 2 CPU
- linux04: 4 CPU
- 總共: 10 CPU

**作業**：
1. Job 2: srun -n 4 -N 4 sleep 120
2. Job 3: srun -n 3 -N 3 sleep 120
3. Job 4: srun -n 1 sleep 120
4. Job 5: srun -n 3 sleep 120

**獨佔模式結果**：
- Job 2 執行（使用 4 節點）
- Job 3, 4, 5 等待
- Job 2 完成後，Job 3 和 Job 4 可同時執行
- Job 5 繼續等待

**可消耗資源結果**：
- Job 2 執行（每節點 1 CPU）
- Job 3 立即執行（使用剩餘 CPU）
- Job 4 立即執行（使用 linux04 剩餘 CPU）
- 3 個作業同時執行

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 RealMemory | 使用 CR_*_Memory 時必須設定 |
| 未指定 --mem | 記憶體環境中必須指定 |
| 同時使用 --oversubscribe 和 --exclusive | 這兩個選項互斥 |
| 期望 -s 在 cons_tres 中有效 | cons_tres 環境不支援 -s |

### 最佳實務

1. **選擇適當的 CR 類型**：
   - 小作業多：CR_CPU_Memory
   - GPU 叢集：CR_Core_Memory + GRES
   - 記憶體敏感：CR_*_Memory

2. **設定合理的預設值**：
   ```
   DefMemPerCPU=2048
   DefCpuPerGPU=4
   DefMemPerGPU=8192
   ```

3. **使用 --exclusive 適時**：
   - 需要整個節點時
   - MPI/OpenMP 混合程式

### 監控資源使用

```bash
# 查看節點資源使用
sinfo -lNe

# 查看作業資源分配
scontrol show job <jobid> | grep -E "NumCPUs|Mem|GRES"

# 查看分區資源
sinfo -o "%P %C %m %G"
```

---

## 快速參考

### slurm.conf 設定

```
# 基本設定
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# GPU 預設值
DefCpuPerGPU=2
DefMemPerGPU=4096

# 節點記憶體
NodeName=node[01-10] CPUs=32 RealMemory=128000 Gres=gpu:4
```

### SelectTypeParameters 選項

| 參數 | 追蹤資源 |
|------|----------|
| CR_CPU | CPU |
| CR_Board | 基板 |
| CR_Socket | 插槽 |
| CR_Core | 核心 |
| CR_Socket_Memory | 插槽 + 記憶體 |
| CR_Core_Memory | 核心 + 記憶體 |
| CR_CPU_Memory | CPU + 記憶體 |

### 作業提交選項

| 選項 | 說明 |
|------|------|
| `--mem=N` | 每節點記憶體 (MB) |
| `--mem-per-cpu=N` | 每 CPU 記憶體 (MB) |
| `--exclusive` | 獨佔節點 |
| `--gpus=N` | GPU 數量 |
| `--gpus-per-node=N` | 每節點 GPU |
| `--cpus-per-gpu=N` | 每 GPU CPU 數 |

### 相關文件

- [GRES 指南](gres.md) - 通用資源設定
- [排程設定](sched_config.md) - 排程器設定
- [cgroups](cgroups.md) - 資源限制
- [slurm.conf](slurm.conf.html) - 主要設定檔
