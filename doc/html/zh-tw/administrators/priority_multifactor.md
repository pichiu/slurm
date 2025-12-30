# Slurm 多因子優先權外掛

---

## TL;DR

Slurm 使用多因子優先權外掛（Multifactor Priority Plugin）計算作業優先權。九個因子影響優先權：年齡（Age）、關聯（Association）、公平共享（Fair-share）、作業大小（Job size）、Nice 值、分割區（Partition）、QOS、站點因子（Site）和 TRES。每個因子有對應的權重（PriorityWeight*），透過加權總和計算最終優先權。使用 `sprio` 指令查看各因子的分解。

---

## Translation（翻譯）

### 目錄

- [介紹](#介紹)
- [多因子作業優先權外掛](#多因子作業優先權外掛)
- [作業優先權因子概述](#作業優先權因子概述)
- [年齡因子](#年齡因子)
- [關聯因子](#關聯因子)
- [作業大小因子](#作業大小因子)
- [Nice 因子](#nice-因子)
- [分割區因子](#分割區因子)
- [服務品質 (QOS) 因子](#服務品質-qos-因子)
- [站點因子](#站點因子)
- [TRES 因子](#tres-因子)
- [公平共享因子](#公平共享因子)
- [sprio 工具](#sprio-工具)
- [配置](#配置)
- [配置範例](#配置範例)

### 介紹

預設情況下，Slurm 設定了 priority/multifactor 外掛，根據多個因子排程作業。

在大多數情況下，使用多因子優先權外掛（Multifactor Priority Plugin）是較佳的選擇，但也可以透過在 slurm.conf 檔案中設定 `PriorityType=priority/basic` 來使用基本的先進先出（FIFO）排程。當 Slurm 由外部排程器控制時，應配置 FIFO 排程。

排程器在做出排程決策時會考慮以下順序來評估作業：

1. 可以搶佔的作業
2. 具有進階預約的作業
3. 分割區 PriorityTier
4. 作業優先權
5. 作業提交時間
6. 作業 ID

這很重要，因為具有最高優先權的作業可能不是排程器首先評估的作業。當有多個作業可以同時評估時（例如請求具有相同 PriorityTier 的分割區的作業），才會考慮作業優先權。

### 多因子作業優先權外掛

多因子作業優先權外掛中有九個因子影響作業優先權：

| 因子 | 說明 |
|------|------|
| **Age（年齡）** | 作業在佇列中等待的時間長度，符合排程條件 |
| **Association（關聯）** | 與每個關聯相關的因子 |
| **Fair-share（公平共享）** | 已承諾的運算資源比例與已消耗資源量之間的差異 |
| **Job size（作業大小）** | 分配給作業的節點或 CPU 數量 |
| **Nice** | 使用者可以控制的因子，用於設定自己作業的優先順序 |
| **Partition（分割區）** | 與每個節點分割區相關的因子 |
| **QOS** | 與每個服務品質相關的因子 |
| **Site（站點）** | 由管理員或站點開發的 job_submit 或 site_factor 外掛決定的因子 |
| **TRES** | 每種 TRES 類型在給定分割區中請求/分配的數量 |

此外，可以為上述每個因子分配權重。這提供了以任何所需比例混合上述任何因子組合的策略能力。例如，站點可以將公平共享配置為主導因子（例如 70%），將作業大小和年齡因子各設定為 15%，並將分割區和 QOS 影響設為零。

### 作業優先權因子概述

作業在任何給定時間的優先權將是 slurm.conf 檔案中已啟用的所有因子的加權總和。作業優先權可以表示為：

```
Job_priority =
    site_factor +
    (PriorityWeightAge) * (age_factor) +
    (PriorityWeightAssoc) * (assoc_factor) +
    (PriorityWeightFairshare) * (fair-share_factor) +
    (PriorityWeightJobSize) * (job_size_factor) +
    (PriorityWeightPartition) * (priority_job_factor) +
    (PriorityWeightQOS) * (QOS_factor) +
    SUM(TRES_weight_cpu * TRES_factor_cpu,
        TRES_weight_<type> * TRES_factor_<type>,
        ...)
    - nice_factor
```

此公式中的所有因子都是從 0.0 到 1.0 的浮點數。權重是無符號的 32 位元整數。作業的優先權是介於 0 到 4294967295 之間的整數。數字越大，作業在佇列中的位置越高，作業越快被排程。作業的優先權及其在佇列中的順序可能會隨時間變化。例如，當 age_weight 非零時，作業在佇列中等待的時間越長，其優先權就會增長得越高。

**重要**：權重值應該足夠高，以獲得良好的有效數字集，因為所有因子都是 0.0 到 1.0 的浮點數。例如，一個作業可能有 .59534 的公平共享因子，另一個作業可能有 .50002 的公平共享因子。如果公平共享權重只設定為 10，兩個作業將具有相同的公平共享優先權。因此，將權重設定得足夠高以避免這種情況，對於要使其佔主導地位的因子，從 1000 左右開始。

### 年齡因子

**注意**：計算年齡因子需要安裝和運作 Slurm 帳務資料庫。

年齡因子（Age Factor）代表作業在佇列中等待並符合執行條件的時間長度。一般來說，作業在佇列中等待的時間越長，其年齡因子就會增長得越大。然而，相依作業的年齡因子在等待其所依賴的作業完成時不會改變。此外，當作業的節點或時間限制超過叢集當前限制而暫停排程時，年齡因子也不會改變。

在某個可配置的時間長度（`PriorityMaxAge`）後，年齡因子將達到最大值 1.0。

### 關聯因子

每個關聯（Association）可以分配一個整數優先權。數字越大，請求此關聯的作業的作業優先權就越高。此優先權值被正規化為所有關聯中最高優先權以成為關聯因子。

### 作業大小因子

作業大小因子（Job Size Factor）與作業請求的節點或 CPU 數量相關。可以根據 slurm.conf 檔案中 `PriorityFavorSmall` 布林值的狀態，配置此因子以有利於較大或較小的作業。

- 當 `PriorityFavorSmall=NO` 時，作業越大，其作業大小因子就越大。請求機器上所有節點的作業將獲得 1.0 的作業大小因子
- 當 `PriorityFavorSmall=YES` 時，單節點作業將獲得 1.0 的作業大小因子

`PriorityFlags` 的 `SMALL_RELATIVE_TO_TIME` 值會修改此行為：CPU 中的作業大小除以時間限制（分鐘），結果再除以系統中 CPU 的總數。

### Nice 因子

使用者可以透過設定作業的 nice 值來調整自己作業的優先權。與系統 nice 類似，正值會負面影響作業的優先權，負值會增加作業的優先權。只有特權使用者可以指定負值。調整範圍是 +/-2147483645。

### 分割區因子

每個節點分割區（Partition）可以分配一個整數優先權。數字越大，請求在此分割區中執行的作業的作業優先權就越高。然後將此優先權值正規化為所有分割區中最高優先權以成為分割區因子。

### 服務品質 (QOS) 因子

每個 QOS 可以分配一個整數優先權。數字越大，請求此 QOS 的作業的作業優先權就越高。然後將此優先權值正規化為所有 QOS 中最高優先權以成為 QOS 因子。

### 站點因子

站點因子（Site Factor）是可以使用 scontrol、透過 job_submit 或 site_factor 外掛設定的因子。一個範例使用案例可能是根據請求多少資源設定特定優先權的 job_submit 外掛。

### TRES 因子

每種 TRES 類型（TRES Type）都有自己的作業優先權因子，代表在給定分割區中請求/分配的 TRES 類型數量。對於全域 TRES 類型（如授權和突發緩衝區），該因子代表整個系統中請求/分配的 TRES 類型數量。作業請求/分配的給定 TRES 類型越多，該作業的作業優先權就越高。

### 公平共享因子

**注意**：計算公平共享因子需要安裝和運作 Slurm 帳務資料庫，以提供下述的分配份額和消耗的運算資源。

作業優先權的公平共享（Fair-share）元件根據已分配的運算資源比例和其作業已消耗的資源來影響使用者排隊作業的排程順序。公平共享因子不涉及固定分配，一旦達到該分配就會切斷使用者對機器的存取。

相反，公平共享因子用於優先排序排隊作業，使那些向服務不足帳戶收費的作業首先被排程，而向服務過度帳戶收費的作業在機器原本會閒置時才被排程。

Slurm 的公平共享因子是介於 0.0 到 1.0 之間的浮點數，反映了使用者已分配的運算資源份額和使用者作業已消耗的運算資源量。值越高，等待排程的作業在佇列中的位置就越高。

預設情況下，運算資源是以 allocated_cpus*seconds 為單位的機器所交付的運算週期。可以透過配置分割區的 `TRESBillingWeights` 選項來考慮其他資源。

**TRESBillingWeights 計費範例：**

```
TRESBillingWeights=CPU=1.0,Mem=0.25G
節點：16CPU, 64GB

           CPUs       Mem GB
Job1: (1 *1.0) + (60*0.25) = (1 + 15) = 16
Job2: (16*1.0) + (1 *0.25) = (16+.25) = 16.25
Job3: (16*1.0) + (60*0.25) = (16+ 15) = 31
```

另一種計算可計費 TRES 的方法是取節點上個別 TRES 的最大值（例如 cpus、mem、gres）加上所有全域 TRES 的總和（例如授權）：

```
           CPUs      Mem GB
Job1: MAX((1 *1.0), (60*0.25)) = 15
Job2: MAX((15*1.0), (1 *0.25)) = 15
Job3: MAX((16*1.0), (64*0.25)) = 16
```

此方法透過在 slurm.conf 中定義 `MAX_TRES` 優先權旗標來啟用。

#### "Fair Tree" 公平共享

從 19.05 版本開始，「Fair Tree」公平共享演算法已成為預設值。

#### "Classic" 公平共享

從 19.05 版本開始，「classic」公平共享演算法不再是預設值，只有在明確配置 `PriorityFlags=NO_FAIR_TREE` 時才會使用。

### sprio 工具

`sprio` 指令提供構成每個作業排程優先權的因子摘要。雖然 `squeue` 有顯示作業綜合優先權的格式選項（%p 和 %Q），但 sprio 可用於顯示每個作業的優先權元件分解。此外，`sprio -w` 選項顯示目前配置的每個因子的權重（PriorityWeightAge、PriorityWeightFairshare 等）。

### 配置

以下 slurm.conf 參數用於配置多因子作業優先權外掛：

| 參數 | 說明 |
|------|------|
| `PriorityType` | 設定為 "priority/multifactor" 以啟用 |
| `PriorityDecayHalfLife` | 決定歷史使用對綜合使用值的貢獻。預設值為 7-0（7 天） |
| `PriorityCalcPeriod` | 重新計算半衰期衰減的時間週期（分鐘）。預設值為 5 |
| `PriorityUsageResetPeriod` | 在此間隔重置關聯的使用量（NONE、NOW、DAILY、WEEKLY、MONTHLY、QUARTERLY、YEARLY） |
| `PriorityFavorSmall` | 設定作業大小因子的極性（YES/NO） |
| `PriorityMaxAge` | 年齡因子達到最大值的佇列等待時間。預設值為 7-0 |
| `PriorityWeightAge` | 年齡因子的權重 |
| `PriorityWeightAssoc` | 關聯因子的權重 |
| `PriorityWeightFairshare` | 公平共享因子的權重 |
| `PriorityWeightJobSize` | 作業大小因子的權重 |
| `PriorityWeightPartition` | 分割區因子的權重 |
| `PriorityWeightQOS` | QOS 因子的權重 |
| `PriorityWeightTRES` | TRES 類型和權重列表 |

**PriorityFlags 選項：**

| 旗標 | 功能 |
|------|------|
| `ACCRUE_ALWAYS` | 無論作業相依性或暫停，都增加優先權年齡因子 |
| `CALCULATE_RUNNING` | 不僅為待處理作業，也為執行中和暫停的作業重新計算優先權 |
| `DEPTH_OBLIVIOUS` | 關聯樹的深度不會負面影響優先權 |
| `NO_FAIR_TREE` | 停用「fair tree」演算法，恢復為「classic」公平共享 |
| `INCR_ONLY` | 優先權值只會增加，永遠不會減少 |
| `MAX_TRES` | 使用 TRES 最大值計算 |
| `NO_NORMAL_ALL` | 設定所有 NO_NORMAL_* 旗標 |
| `NO_NORMAL_ASSOC` | 關聯因子不正規化 |
| `NO_NORMAL_PART` | 分割區因子不正規化 |
| `NO_NORMAL_QOS` | QOS 因子不正規化 |
| `NO_NORMAL_TRES` | TRES 因子不正規化 |
| `SMALL_RELATIVE_TO_TIME` | 作業大小基於大小除以時間限制 |

### 配置範例

**使用衰減的範例：**

```bash
# 啟用多因子作業優先權外掛（含衰減）
PriorityType=priority/multifactor

# 2 週半衰期
PriorityDecayHalfLife=14-0

# 作業越大，其作業大小優先權越高
PriorityFavorSmall=NO

# 在佇列中等待 2 週後，作業的年齡因子達到 1.0
PriorityMaxAge=14-0

# 權重配置
PriorityWeightAge=1000
PriorityWeightFairshare=10000
PriorityWeightJobSize=1000
PriorityWeightPartition=1000
PriorityWeightQOS=0  # 不使用 QOS 因子
```

**不使用衰減的範例：**

```bash
# 啟用多因子作業優先權外掛
PriorityType=priority/multifactor

# 不套用衰減
PriorityDecayHalfLife=0

# 每月重置使用量
PriorityUsageResetPeriod=MONTHLY

# 作業越大，其作業大小優先權越高
PriorityFavorSmall=NO

# 在佇列中等待 2 週後，作業的年齡因子達到 1.0
PriorityMaxAge=14-0

# 權重配置
PriorityWeightAge=1000
PriorityWeightFairshare=10000
PriorityWeightJobSize=1000
PriorityWeightPartition=1000
PriorityWeightQOS=0  # 不使用 QOS 因子
```

---

## Explanation（解釋）

### 優先權計算流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    作業優先權計算流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  作業提交 ──► 初始優先權計算 ──► 進入佇列                        │
│                     │                                           │
│                     ▼                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               優先權因子 (0.0 - 1.0)                     │   │
│  │                                                         │   │
│  │  Age Factor ─────────────┐                              │   │
│  │  Assoc Factor ───────────┤                              │   │
│  │  Fairshare Factor ───────┤    ┌───────────────────┐    │   │
│  │  Job Size Factor ────────┼───►│ 加權總和計算       │    │   │
│  │  Nice Factor ────────────┤    │                   │    │   │
│  │  Partition Factor ───────┤    │ Σ(Weight × Factor)│    │   │
│  │  QOS Factor ─────────────┤    └─────────┬─────────┘    │   │
│  │  Site Factor ────────────┤              │              │   │
│  │  TRES Factors ───────────┘              ▼              │   │
│  │                                   最終優先權            │   │
│  │                               (0 - 4294967295)         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  優先權隨時間變化：                                              │
│  • Age Factor 增加（等待越久越高）                              │
│  • Fairshare Factor 變化（根據使用量）                          │
└─────────────────────────────────────────────────────────────────┘
```

### 公平共享的概念

```
          資源分配                     實際使用
    ┌──────────────────┐        ┌──────────────────┐
    │                  │        │                  │
    │    帳戶 A: 50%   │        │    帳戶 A: 30%   │ ◄── 使用不足
    │                  │        │                  │      優先權 ↑
    ├──────────────────┤        ├──────────────────┤
    │                  │        │                  │
    │    帳戶 B: 30%   │        │    帳戶 B: 50%   │ ◄── 使用過度
    │                  │        │                  │      優先權 ↓
    ├──────────────────┤        ├──────────────────┤
    │                  │        │                  │
    │    帳戶 C: 20%   │        │    帳戶 C: 20%   │ ◄── 使用正常
    │                  │        │                  │
    └──────────────────┘        └──────────────────┘

    公平共享確保長期來看，每個帳戶獲得其承諾的資源份額
```

---

## Practical Example（實用範例）

### 範例 1：查看作業優先權分解

```bash
# 查看所有待處理作業的優先權因子
sprio

# 輸出範例：
#   JOBID PARTITION   PRIORITY       SITE        AGE  FAIRSHARE    JOBSIZE  PARTITION        QOS
#   12345 batch          50000          0       1000       8500        500       5000      35000
#   12346 gpu            45000          0        800       7000        200      12000      25000
```

**欄位說明：**
- `PRIORITY`：最終計算的優先權值
- `SITE`：站點因子貢獻
- `AGE`：年齡因子貢獻
- `FAIRSHARE`：公平共享因子貢獻
- `JOBSIZE`：作業大小因子貢獻
- `PARTITION`：分割區因子貢獻
- `QOS`：QOS 因子貢獻

### 範例 2：查看優先權權重配置

```bash
# 顯示目前配置的權重
sprio -w

# 輸出範例：
#   JOBID PARTITION   PRIORITY       SITE        AGE  FAIRSHARE    JOBSIZE  PARTITION        QOS
#   Weights                             1      10000     100000      10000      10000      50000
```

### 範例 3：查看特定使用者的作業優先權

```bash
# 查看特定使用者的優先權
sprio -u alice

# 使用長格式查看更多詳細資訊
sprio -l -u alice
```

### 範例 4：使用 nice 值調整優先權

```bash
# 提交作業時降低優先權（nice 正值）
sbatch --nice=1000 job.sh

# 提高優先權（需要特權，nice 負值）
sbatch --nice=-500 job.sh

# 修改已提交作業的 nice 值
scontrol update jobid=12345 nice=500
```

### 範例 5：配置分割區優先權

```bash
# 在 slurm.conf 中設定分割區優先權
# PartitionName=high PriorityJobFactor=100 ...
# PartitionName=normal PriorityJobFactor=50 ...
# PartitionName=low PriorityJobFactor=10 ...

# 查看分割區配置
scontrol show partition | grep -E "PartitionName|PriorityJobFactor"
```

### 範例 6：配置 QOS 優先權

```bash
# 新增具有優先權的 QOS
sacctmgr add qos high priority=1000
sacctmgr add qos normal priority=500
sacctmgr add qos low priority=100

# 查看 QOS 優先權
sacctmgr show qos format=name,priority

# 提交作業時指定 QOS
sbatch --qos=high job.sh
```

### 範例 7：查看公平共享狀態

```bash
# 查看公平共享資訊
sshare -a

# 輸出範例：
#   Account       User  RawShares  NormShares    RawUsage  EffectvUsage  FairShare
#   root                              1.000000       12345      1.000000
#   research                    100   0.500000        6000      0.486000
#   research     alice       50      0.250000        2000      0.162000   0.650000
#   research     bob         50      0.250000        4000      0.324000   0.400000
```

### 範例 8：完整優先權配置

```bash
# slurm.conf 優先權配置範例
cat << 'EOF' >> /etc/slurm/slurm.conf
# 優先權類型
PriorityType=priority/multifactor

# 衰減設定
PriorityDecayHalfLife=14-0
PriorityCalcPeriod=5

# 年齡因子設定
PriorityMaxAge=7-0

# 作業大小偏好
PriorityFavorSmall=NO

# 權重配置
PriorityWeightAge=10000
PriorityWeightAssoc=0
PriorityWeightFairshare=50000
PriorityWeightJobSize=5000
PriorityWeightPartition=10000
PriorityWeightQOS=20000

# TRES 權重
PriorityWeightTRES=CPU=1000,Mem=500,GRES/gpu=5000

# 優先權旗標
PriorityFlags=CALCULATE_RUNNING
EOF
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| 權重設定過低 | 因子之間差異無法區分 | 設定權重至少 1000 以上 |
| 未啟用帳務 | Fair-share 和 Age 因子無法計算 | 配置 SlurmDBD |
| 所有權重為 0 | 所有作業優先權相同 | 至少設定一個非零權重 |
| PriorityMaxAge 太短 | 作業快速達到最大年齡優先權 | 根據平均等待時間調整 |
| 未設定 PriorityDecayHalfLife | 使用量永不衰減 | 設定適當的衰減週期 |

### ✅ 實用技巧

1. **平衡權重設定**
   ```bash
   # 建議的權重比例範例
   # Fair-share 佔主導：70%
   # Age 和 JobSize：各 15%
   PriorityWeightFairshare=70000
   PriorityWeightAge=15000
   PriorityWeightJobSize=15000
   ```

2. **監控優先權變化**
   ```bash
   # 定期檢查優先權分佈
   sprio | awk '{print $3}' | sort -n | uniq -c
   ```

3. **除錯優先權問題**
   ```bash
   # 查看特定作業的詳細優先權
   sprio -j 12345 -l

   # 比較兩個作業
   sprio -j 12345,12346
   ```

4. **調整公平共享**
   ```bash
   # 查看帳戶的份額配置
   sacctmgr show assoc format=account,user,share

   # 修改份額
   sacctmgr modify account research set fairshare=100
   ```

5. **使用 TRES 權重**
   ```bash
   # 讓 GPU 作業獲得更高優先權
   PriorityWeightTRES=CPU=100,GRES/gpu=10000
   ```

---

## Quick Reference（快速參考）

### 優先權因子

| 因子 | 範圍 | 說明 |
|------|------|------|
| Age | 0.0 - 1.0 | 等待時間 |
| Association | 0.0 - 1.0 | 關聯優先權 |
| Fair-share | 0.0 - 1.0 | 公平共享 |
| Job Size | 0.0 - 1.0 | 作業大小 |
| Nice | -2147483645 - +2147483645 | 使用者調整 |
| Partition | 0.0 - 1.0 | 分割區優先權 |
| QOS | 0.0 - 1.0 | QOS 優先權 |
| Site | 整數 | 站點自訂 |
| TRES | 0.0 - 1.0 | TRES 請求 |

### slurm.conf 參數

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `PriorityType` | priority/multifactor | 優先權外掛類型 |
| `PriorityDecayHalfLife` | 7-0 | 使用量衰減半衰期 |
| `PriorityCalcPeriod` | 5 | 重新計算週期（分鐘） |
| `PriorityMaxAge` | 7-0 | 年齡因子最大值時間 |
| `PriorityFavorSmall` | NO | 是否偏好小作業 |
| `PriorityUsageResetPeriod` | NONE | 使用量重置週期 |

### sprio 指令

| 指令 | 功能 |
|------|------|
| `sprio` | 顯示所有待處理作業優先權 |
| `sprio -j JOBID` | 顯示特定作業優先權 |
| `sprio -u USER` | 顯示特定使用者作業優先權 |
| `sprio -w` | 顯示權重配置 |
| `sprio -l` | 長格式輸出 |
| `sprio -n` | 不顯示標題 |

### sshare 指令

| 指令 | 功能 |
|------|------|
| `sshare` | 顯示公平共享資訊 |
| `sshare -a` | 顯示所有關聯 |
| `sshare -A ACCOUNT` | 顯示特定帳戶 |
| `sshare -u USER` | 顯示特定使用者 |
| `sshare -l` | 長格式輸出 |
