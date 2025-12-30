# Slurm 共享可消耗資源 (Sharing Consumable Resources)

## TL;DR

可消耗資源共享由分割區的 `OverSubscribe` 設定控制。`OverSubscribe=NO` 不共享；`OverSubscribe=YES` 允許作業請求共享；`OverSubscribe=FORCE` 強制共享（預設每資源最多 4 個作業）。使用 `select/cons_tres` 時，共享單位取決於 `SelectTypeParameters`（Core/CPU/Socket）。記憶體可透過 `CR_*_Memory` 設定追蹤以避免交換。

---

## 翻譯

### CPU 管理

（免責聲明：在此「CPU 管理」章節中，「可消耗資源」一詞不包含記憶體。記憶體作為可消耗資源的管理在下方獨立章節討論。）

分割區的 `OverSubscribe` 設定適用於**被選擇進行排程的實體**：

- 當啟用 `select/linear` 外掛程式時，分割區的 `OverSubscribe` 設定控制**節點**是否在作業之間共享
- 當啟用預設的 `select/cons_tres` 外掛程式時，分割區的 `OverSubscribe` 設定控制**設定的可消耗資源**是否在作業之間共享。當可消耗資源（如核心、socket 或 CPU）被共享時，表示可以有多個作業被分配到它

---

### 共享行為對照表

| 選擇設定 | OverSubscribe 設定 | 結果行為 |
|----------|-------------------|----------|
| **select/linear** | NO | 整個節點分配給作業。每個分割區/佇列的每個節點不會執行超過一個作業 |
| | YES | 預設同 NO。如果每個作業都透過 `srun --oversubscribe` 選項允許共享，則分配給作業的節點可以與其他作業共享 |
| | FORCE | 每個整個節點可以分配給多個作業，最多到指定數量（預設每節點 4 個作業）|
| **select/cons_tres** + **CR_Core**/_Memory | NO | 核心分配給作業。每個分割區/佇列的每個核心不會執行超過一個作業 |
| | YES | 預設同 NO。如果每個作業都允許共享，則分配給作業的核心可以與其他作業共享 |
| | FORCE | 每個核心可以分配給多個作業，最多到指定數量（預設每核心 4 個作業）|
| **select/cons_tres** + **CR_CPU**/_Memory | NO | CPU 分配給作業。每個分割區/佇列的每個 CPU 不會執行超過一個作業 |
| | YES | 預設同 NO。如果每個作業都允許共享，則分配給作業的 CPU 可以與其他作業共享 |
| | FORCE | 每個 CPU 可以分配給多個作業，最多到指定數量（預設每 CPU 4 個作業）|
| **select/cons_tres** + **CR_Socket**/_Memory | NO | Socket 分配給作業。每個分割區/佇列的每個 Socket 不會執行超過一個作業 |
| | YES | 預設同 NO。如果每個作業都允許共享，則分配給作業的 Socket 可以與其他作業共享 |
| | FORCE | 每個 Socket 可以分配給多個作業，最多到指定數量（預設每 Socket 4 個作業）|

---

### 最少負載演算法

當設定 `OverSubscribe=FORCE` 時，可消耗資源使用**最少負載**演算法排程作業：

1. 空閒的 CPU/核心/socket 會在忙碌的之前分配給作業
2. 執行一個作業的 CPU/核心/socket 會在執行兩個或更多作業的之前分配給作業

**粒度差異**：

這是區分 `select/cons_tres` 和 `select/linear` 外掛程式的關鍵：

- `select/cons_tres`：節點的 CPU 不會被過度承諾，直到**所有**其他節點上的 CPU 都被過度承諾
- `select/linear`：只計算節點上的作業數量，不追蹤每個節點的 CPU 使用情況

**範例**：如果一個作業分配了節點一半的 CPU，然後提交的第二個作業需要超過一半的 CPU，可消耗資源外掛程式會嘗試將新作業放置在其他有超過一半 CPU 可用的忙碌節點上。

---

### FORCE 數量語法

`select/cons_tres` 外掛程式支援 `OverSubscribe=FORCE:<num>` 語法：

| 設定 | 行為 |
|------|------|
| `OverSubscribe=FORCE:3` + `CR_Core` | 每個核心最多執行 3 個作業 |
| `OverSubscribe=FORCE:3` + `CR_Socket` | 每個 socket 最多執行 3 個作業 |

---

### 多分割區中的節點

Slurm 自 0.7.0 版本起支援將節點設定在多個分割區中。

#### 共享作業定義

| 類型 | 定義 |
|------|------|
| 可共享作業 | 提交到 `OverSubscribe=FORCE` 分割區，或 `OverSubscribe=YES` 分割區且作業請求了 `srun --oversubscribe` |
| 不可共享作業 | 提交到 `OverSubscribe=NO` 分割區，或 `OverSubscribe=YES` 分割區且作業**未**請求共享資源 |

#### 共享行為矩陣

| | 第一個作業可共享 | 第一個作業不可共享 |
|--|-----------------|-------------------|
| **第二個作業可共享** | 兩個作業可在相同節點執行並可共享資源 | 作業不在相同節點執行 |
| **第二個作業不可共享** | 作業不在相同節點執行 | 作業可在相同節點執行但不共享資源 |

---

### 多分割區場景

| Slurm 設定 | 結果行為 |
|------------|----------|
| 兩個 `OverSubscribe=NO` 分割區指派相同節點 | 來自任一分割區的作業將分配到所有可用的可消耗資源。不共享資源。一個節點可能有 2 個作業執行，每個作業可來自不同分割區 |
| 相同節點指派給 `OverSubscribe=FORCE` 和 `OverSubscribe=NO` 分割區 | 節點一次只執行來自一個分割區的作業。如果執行 NO 分割區的作業，則不共享資源；如果執行 FORCE 分割區的作業，則可共享資源 |
| 兩個 `OverSubscribe=FORCE` 分割區指派相同節點 | 來自任一分割區的作業將被分配可消耗資源。所有資源可共享。一個節點可能有 2 個作業執行，每個作業可來自不同分割區 |
| `OverSubscribe=FORCE:3` 和 `OverSubscribe=FORCE:5` 分割區指派相同節點 | 類似上述行為。但沒有可消耗資源會從第一個分割區執行超過 3 個作業，從第二個分割區執行超過 5 個作業。一個可消耗資源一次最多可能有 8 個作業執行 |

**飢餓問題警告**：

混合共享設定（如 FORCE + NO）可能導致分割區之間的作業**飢餓**：

- 如果節點正在執行 NO 分割區的作業，即使 FORCE 分割區的作業有更高優先順序，這些節點也只能繼續用於該分割區
- FORCE 分割區的作業更容易長時間佔用節點，因為可消耗資源「共享」為新作業提供更多資源可用性

---

### 記憶體管理

記憶體作為可消耗資源的管理保持不變，可用於防止記憶體過度訂閱（導致記憶體頁面交換和嚴重效能降低）。

| 選擇設定 | 結果行為 |
|----------|----------|
| `select/linear` | 不追蹤記憶體分配。作業分配到節點時不考慮是否有足夠的可用記憶體。可能發生交換！|
| `select/linear` + `CR_Memory` | 追蹤記憶體分配。沒有足夠可用記憶體滿足作業需求的節點不會被分配給作業 |
| `select/cons_tres` + `CR_Core`/`CR_CPU`/`CR_Socket` | 不追蹤記憶體分配。可能發生交換！|
| `select/cons_tres` + `CR_Core_Memory`/`CR_CPU_Memory`/`CR_Socket_Memory` | 追蹤所有作業的記憶體分配。沒有足夠可用記憶體的節點不會被分配給作業 |

---

### 指定記憶體需求

使用者可以用兩種方式指定作業的記憶體需求：

| 選項 | 說明 | 建議外掛程式 |
|------|------|--------------|
| `--mem=<num>` | 每分配節點的記憶體需求 | select/linear |
| `--mem-per-cpu=<num>` | 每分配 CPU 的記憶體需求 | select/cons_tres |

---

### 記憶體設定參數

系統管理員可使用以下 slurm.conf 選項設定記憶體的預設值和最大值：

| 參數 | 說明 |
|------|------|
| `DefMemPerCPU` | 每 CPU 預設記憶體 |
| `DefMemPerNode` | 每節點預設記憶體 |
| `MaxMemPerCPU` | 每 CPU 最大記憶體 |
| `MaxMemPerNode` | 每節點最大記憶體 |

使用者可在提交作業時使用 `--mem` 或 `--mem-per-cpu` 選項覆寫預設值，但不能超過最大值。

---

### 記憶體強制執行

記憶體分配的強制執行透過以下方式執行：

1. 在啟動任務前將「最大資料段大小」和「最大虛擬記憶體大小」系統限制設定為適當值
2. 由計費外掛程式管理，定期收集執行中作業的資料

設定 `JobAcctGather` 和 `JobAcctFrequency` 為適合您系統的值。

**注意**：`--oversubscribe` 和 `--exclusive` 選項在作業提交時是互斥的。如果提交作業時同時設定兩個選項，提交命令將會失敗。

---

## 說明

### 資源共享概念圖

```
OverSubscribe=NO:
┌─────────────────┐
│ Core 0: Job A   │  每個核心只執行一個作業
│ Core 1: Job B   │
│ Core 2: (空閒)  │
│ Core 3: (空閒)  │
└─────────────────┘

OverSubscribe=FORCE:
┌─────────────────┐
│ Core 0: Job A,B │  核心可執行多個作業
│ Core 1: Job C   │
│ Core 2: Job D,E │
│ Core 3: (空閒)  │
└─────────────────┘
```

---

## 實務範例

### 基本共享設定

```
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# 不共享的分割區
PartitionName=exclusive Nodes=node[01-10] OverSubscribe=NO

# 允許共享的分割區
PartitionName=shared Nodes=node[11-20] OverSubscribe=YES

# 強制共享的分割區
PartitionName=timeshare Nodes=node[21-30] OverSubscribe=FORCE:4
```

### 高吞吐量設定

```
# slurm.conf - 短作業時間片輪轉
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
PreemptMode=GANG

PartitionName=short Nodes=ALL OverSubscribe=FORCE:8
```

### 記憶體保護設定

```
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

DefMemPerCPU=2000
MaxMemPerCPU=8000

PartitionName=default Nodes=ALL OverSubscribe=FORCE
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未追蹤記憶體導致交換 | 使用 `CR_*_Memory` 設定 |
| 混合 NO/FORCE 分割區造成飢餓 | 避免混合設定或接受其後果 |
| FORCE 數量過高 | 根據負載調整，避免過度競爭 |
| 忽略記憶體限制 | 設定 DefMemPerCPU 和 MaxMemPerCPU |

### 建議

1. **記憶體追蹤**：
   - 總是使用 `_Memory` 後綴的 SelectTypeParameters
   - 避免記憶體交換造成的效能問題

2. **FORCE 數量調校**：
   - 從預設值 4 開始
   - 監控系統負載後調整

3. **多分割區設計**：
   - 盡量避免混合共享設定
   - 如需混合，了解飢餓風險

---

## 快速參考

### slurm.conf 設定

```
# 資源選擇
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# 分割區共享設定
PartitionName=name OverSubscribe=[NO|YES|FORCE[:num]]

# 記憶體設定
DefMemPerCPU=2000
MaxMemPerCPU=8000
```

### OverSubscribe 選項

| 選項 | 說明 |
|------|------|
| NO | 不共享資源 |
| YES | 允許作業請求共享 |
| FORCE | 強制共享（預設最多 4 個作業）|
| FORCE:N | 強制共享，每資源最多 N 個作業 |

### 使用者選項

| 選項 | 說明 |
|------|------|
| `--oversubscribe` | 請求共享資源 |
| `--exclusive` | 請求獨佔資源 |
| `--mem=N` | 每節點記憶體需求 |
| `--mem-per-cpu=N` | 每 CPU 記憶體需求 |

### 相關文件

- [可消耗資源](cons_tres.md) - 資源選擇設定
- [Gang 排程](gang_scheduling.md) - 時間片輪轉
- [搶佔](preempt.md) - 作業搶佔設定
