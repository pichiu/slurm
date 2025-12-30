# Slurm Gang 排程 (Gang Scheduling)

## TL;DR

Gang 排程允許多個作業分配到相同資源，透過時間片輪轉 (timeslicing) 交替執行。啟用方式：`PreemptMode=GANG` + `OverSubscribe=FORCE`。時間片間隔由 `SchedulerTimeSlice` 控制（預設 30 秒）。需注意記憶體設定以避免交換。此功能可提高響應性和利用率，允許短作業更快開始執行。

---

## 翻譯

### 概觀

Slurm 支援時間片 Gang 排程，允許兩個或多個作業分配到同一分割區的相同資源，這些作業會交替暫停以讓一個作業在設定的時間段內獨佔存取資源。

Slurm 也支援搶佔式作業排程，允許較高 `PriorityTier` 分割區或搶佔 QOS 中的作業搶佔其他作業。[搶佔](preempt.md)與 Gang 排程相關，因為 SUSPEND 是其中一種搶佔模式，它使用 Gang 排程器來恢復暫停的作業。

**時間片排程的好處**：
- 提高響應性和利用率
- 允許更多作業更快開始執行
- 短執行時間的作業不需要在長執行時間的作業後等待
- 可以與長執行時間的作業「並行」執行
- 過度承諾資源提供「本地回填」機會

**Gang 排程運作方式**：
1. Gang 排程邏輯在每個分割區獨立運作
2. 如果新作業分配到已分配給現有作業的資源，外掛程式會暫停新作業
3. 等到設定的 `SchedulerTimeslice` 間隔後
4. 暫停正在執行的作業，讓新作業使用資源
5. 持續輪轉直到其中一個作業終止

**注意**：異質作業 (Heterogeneous jobs) 不包含在 Gang 排程操作中。

---

### 設定

#### 重要設定參數

| 參數 | 說明 |
|------|------|
| **SelectType** | 支援 `select/linear` 或 `select/cons_tres` |
| **SelectTypeParameters** | 建議追蹤記憶體使用量 |
| **DefMemPerCPU** | 預設每 CPU 記憶體或 DefMemPerNode |
| **JobAcctGatherType** | 如需強制記憶體限制 |
| **PreemptMode** | 設定 GANG 選項 |
| **SchedulerTimeSlice** | 時間片間隔（秒），預設 30 |
| **OverSubscribe** | 設定為 FORCE |

#### SelectTypeParameters 建議

| SelectType | 建議設定 |
|------------|----------|
| select/linear | `CR_Memory` |
| select/cons_tres | `CR_Core_Memory` 或包含 Memory |

由於資源會被過度分配（暫停的作業仍在記憶體中），必須追蹤記憶體使用量以避免記憶體頁面交換。

#### 記憶體設定

```
# slurm.conf
DefMemPerCPU=2000      # 或使用 DefMemPerNode
MaxMemPerCPU=4000      # 可選
```

所有作業必須能同時容納在記憶體中才能進行 Gang 排程。

#### PreemptMode 設定

```
# slurm.conf
PreemptMode=GANG
```

**注意**：
- Gang 排程在每個分割區獨立執行
- 如果只想要時間片輪轉而不要搶佔，不建議設定重疊節點的分割區
- 如果要使用 `PreemptType=preempt/partition_prio`，需要重疊分割區和 `PreemptMode=SUSPEND,GANG`
- 不同分割區之間不會發生時間片輪轉

#### OverSubscribe 設定

```
# slurm.conf
PartitionName=active OverSubscribe=FORCE    # 預設最多 4 個作業共享
PartitionName=active OverSubscribe=FORCE:6  # 最多 6 個作業共享
PartitionName=active OverSubscribe=FORCE:2  # 最多 2 個作業共享
```

---

### 時間片器設計與操作

**運作機制**：

1. Gang 排程器追蹤分配給所有作業的資源
2. 為每個分割區維護「活動位元圖」(active bitmap)
3. 新作業分配資源時，與活動位元圖比較：
   - 資源不重疊：加入活動位元圖
   - 資源重疊：暫停新作業

**時間片器執行緒**：

1. 以設定的 `SchedulerTimeSlice` 間隔休眠
2. 喚醒時檢查每個分割區的暫停作業
3. 如果發現暫停作業：
   - 將所有執行中作業移到作業佇列末端
   - 從等待最久的暫停作業開始重建活動位元圖
   - 暫停不再是活動位元圖一部分的作業
   - 恢復新加入活動位元圖的作業

**演算法目標**：
- 防止作業飢餓（無限期保持暫停狀態）
- 盡可能公平分配執行時間
- 保持所有資源忙碌

**觀察時間片操作**：
```bash
squeue -i<time>  # time 設為 SchedulerTimeSlice 值
```

---

### 簡單範例

**叢集設定**：
```
PARTITION AVAIL  TIMELIMIT NODES  STATE NODELIST
active*      up   infinite     5   idle n[12-16]

PreemptMode             = GANG
SchedulerTimeSlice      = 30
SchedulerType           = sched/builtin
```

**提交兩個作業**：
```bash
$ sbatch -N5 ./myload 300
sbatch: Submitted batch job 3

$ sbatch -N5 ./myload 300
sbatch: Submitted batch job 4

$ squeue
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
    3    active  myload  user  R  0:13     5 n[12-16]
    4    active  myload  user  S  0:00     5 n[12-16]
```

**30 秒後交換**：
```bash
$ squeue
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
    4    active  myload  user  R  0:08     5 n[12-16]
    3    active  myload  user  S  0:41     5 n[12-16]
```

**可能的副作用**：

立即被暫停的作業可能產生以下輸出：
```
srun: Job step creation temporarily disabled, retrying
srun: Job step creation still disabled, retrying
srun: Job step created
```

這是因為 srun 嘗試在暫停的分配中啟動作業步驟。當 Gang 排程器啟用時，這種輸出應視為正常。

---

### 更多範例

#### 保持資源忙碌

```bash
$ sbatch -N3 ./myload 300   # Job 9
$ sbatch -N2 ./myload 300   # Job 10
$ sbatch -N3 ./myload 300   # Job 11

$ squeue
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
    9    active  myload  user  R  0:11     3 n[12-14]
   10    active  myload  user  R  0:08     2 n[15-16]
   11    active  myload  user  S  0:00     3 n[12-14]
```

作業 10 持續執行，而作業 9 和 11 進行時間片輪轉。

#### 本地回填

```bash
$ sbatch -N3 ./myload 300   # Job 12
$ sbatch -N5 ./myload 300   # Job 13
$ sbatch -N2 ./myload 300   # Job 14

$ squeue
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
   12    active  myload  user  R  0:14     3 n[12-14]
   14    active  myload  user  R  0:06     2 n[15-16]
   13    active  myload  user  S  0:00     5 n[12-16]
```

沒有時間片和回填排程器，作業 14 必須等待作業 13 完成。

**本地回填**僅發生在佇列中足夠接近的作業之間，由 `OverSubscribe=FORCE:max_share` 控制範圍。

---

### 可消耗資源範例

#### CR_Core_Memory vs CR_CPU_Memory

**CR_Core_Memory**：
- 選擇器將 CPU 視為**特定**分配給作業的個別資源
- 作業「固定」到特定核心
- 支援 CPU 綁定

**CR_CPU_Memory**：
- 選擇器將 CPU 視為簡單、**可互換**的計算資源
- 依賴 Linux 核心在核心之間移動作業
- 不支援 CPU 綁定

#### CR_Core_Memory 範例

```
SelectTypeParameters    = CR_CORE_MEMORY
```

提交 6 個作業（每個請求 2 CPU/節點，節點有 8 CPU）：

```bash
$ squeue
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
   44    active  myload  user  R  0:09     5 n[12-16]
   45    active  myload  user  R  0:08     5 n[12-16]
   46    active  myload  user  R  0:08     5 n[12-16]
   47    active  myload  user  R  0:07     5 n[12-16]
   48    active  myload  user  S  0:00     5 n[12-16]
   49    active  myload  user  S  0:00     5 n[12-16]
```

一段時間後：
```bash
$ squeue  # 作業 46、47 持續執行（不共享核心）
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
   44    active  myload  user  R  1:23     5 n[12-16]
   45    active  myload  user  R  1:22     5 n[12-16]
   46    active  myload  user  R  2:22     5 n[12-16]  # 持續執行
   47    active  myload  user  R  2:21     5 n[12-16]  # 持續執行
   48    active  myload  user  S  1:00     5 n[12-16]
   49    active  myload  user  S  1:00     5 n[12-16]
```

作業 46 和 47 持續執行，因為它們不與其他作業共享核心。

#### CR_CPU_Memory 範例

```
SelectTypeParameters    = CR_CPU_MEMORY
```

所有 6 個作業共享 CPU 時間：
```bash
$ squeue  # 所有作業執行時間大致相等
JOBID PARTITION    NAME  USER ST  TIME NODES NODELIST
   51    active  myload  user  R  3:18     5 n[12-16]
   52    active  myload  user  R  3:18     5 n[12-16]
   53    active  myload  user  R  3:17     5 n[12-16]
   54    active  myload  user  R  3:16     5 n[12-16]
   55    active  myload  user  S  3:00     5 n[12-16]
   56    active  myload  user  S  3:00     5 n[12-16]
```

---

### 手動暫停注意事項

手動暫停作業（`scontrol suspend ...`）會釋放其 CPU 供其他作業分配。恢復先前暫停的作業可能導致多個作業分配到相同 CPU，這會觸發 Gang 排程。

使用 `scancel` 發送 SIGSTOP 和 SIGCONT 訊號可以停止作業而不釋放 CPU，在許多情況下是更好的機制。

---

## 說明

### Gang 排程 vs 回填排程

| 特性 | Gang 排程 | 回填排程 |
|------|-----------|----------|
| 資源使用 | 過度承諾 | 不過度承諾 |
| 記憶體需求 | 所有作業需同時容納 | 依序執行 |
| 短作業優勢 | 可更快開始 | 依賴空隙填補 |
| 複雜度 | 較高 | 中等 |

### 時間片概念圖

```
時間 →
作業 A: ████████░░░░░░░░████████░░░░░░░░████████
作業 B: ░░░░░░░░████████░░░░░░░░████████░░░░░░░░
        ←--30s--→←--30s--→←--30s--→←--30s--→
                  時間片輪轉
```

---

## 實務範例

### 基本 Gang 排程設定

```
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
PreemptMode=GANG
SchedulerTimeSlice=30
DefMemPerCPU=2000

PartitionName=gang Nodes=node[01-10] OverSubscribe=FORCE:4
```

### 搭配搶佔的設定

```
# slurm.conf
PreemptMode=SUSPEND,GANG
PreemptType=preempt/partition_prio

PartitionName=high PriorityTier=100 Nodes=node[01-10] OverSubscribe=FORCE
PartitionName=low  PriorityTier=50  Nodes=node[01-10] OverSubscribe=FORCE
```

### 監控 Gang 排程

```bash
# 即時觀察時間片輪轉
squeue -i 30

# 查看作業狀態變化
watch -n 5 'squeue -o "%.10i %.9P %.8j %.8u %.2t %.10M %.6D %R"'

# 查看設定
scontrol show config | grep -E "PreemptMode|SchedulerTimeSlice|OverSubscribe"
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定記憶體追蹤 | 使用 CR_Memory 或 CR_Core_Memory |
| OverSubscribe 未設為 FORCE | 必須設定 FORCE 才能觸發時間片 |
| 未設定 DefMemPerCPU | 設定預設記憶體以確保所有作業能同時容納 |
| 在重疊分割區使用純 Gang | 考慮是否需要搶佔或避免重疊 |
| SchedulerTimeSlice 過短 | 會增加開銷，建議至少 30 秒 |

### 調校建議

1. **SchedulerTimeSlice**：
   - 過短：增加排程開銷
   - 過長：降低響應性
   - 建議：30-120 秒

2. **OverSubscribe max_share**：
   - 較高值：更多本地回填機會
   - 較低值：減少記憶體壓力
   - 建議：2-4

3. **記憶體規劃**：
   - 確保 `max_share * 每作業記憶體 <= 節點記憶體`
   - 使用 cgroup 限制記憶體使用

---

## 快速參考

### slurm.conf 設定

```
# Gang 排程基本設定
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
PreemptMode=GANG
SchedulerTimeSlice=30
DefMemPerCPU=2000

# 分割區設定
PartitionName=gang Nodes=ALL OverSubscribe=FORCE:4
```

### 作業狀態對照

| 狀態 | 符號 | 說明 |
|------|------|------|
| RUNNING | R | 正在執行 |
| SUSPENDED | S | 被 Gang 排程器暫停 |

### 常用命令

| 命令 | 功能 |
|------|------|
| `squeue -i 30` | 每 30 秒顯示佇列（觀察時間片）|
| `scontrol suspend <jobid>` | 手動暫停作業 |
| `scontrol resume <jobid>` | 手動恢復作業 |
| `scontrol show config \| grep TimeSlice` | 查看時間片設定 |

### 相關文件

- [搶佔](preempt.md) - 作業搶佔機制
- [可消耗資源](cons_tres.md) - 資源選擇外掛程式
- [排程設定](sched_config.md) - 排程器設定
- [QoS](qos.md) - 服務品質設定
