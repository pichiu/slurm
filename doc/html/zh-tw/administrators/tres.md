# Slurm 可追蹤資源 (TRES - Trackable RESources)

## TL;DR

TRES 是可追蹤使用量或用於強制限制的資源。TRES 由類型和名稱組成，預設類型包括 CPU、Memory、Node、Energy 等。使用 `AccountingStorageTRES` 設定追蹤的 TRES，`TRESBillingWeights` 設定計費權重，`PriorityWeightTRES` 設定優先順序權重。計費公式預設為各 TRES 加權總和。

---

## 翻譯

### 概觀

TRES 是可以追蹤使用量或用於強制限制的資源。TRES 是類型 (Type) 和名稱 (Name) 的組合。類型是預先定義的。

### 目前的 TRES 類型

| 類型 | 說明 |
|------|------|
| BB | 突發緩衝 (Burst Buffers) |
| Billing | 計費 |
| CPU | 處理器 |
| Energy | 能源 |
| FS | 檔案系統 |
| GRES | 通用資源 |
| IC | 互連網路 |
| License | 授權 |
| Mem | 記憶體 |
| Node | 節點 |
| Pages | 頁面 |
| VMem | 虛擬記憶體/大小 |

**特殊說明**：
- **Billing TRES**：從分割區的 TRESBillingWeights 計算
- **FS TRES**：有效值為 'disk'（本地磁碟）和 'lustre'，主要用於報告使用量而非限制存取
- **IC TRES**：有效值為 'ofed'，主要用於報告使用量而非限制存取

---

### slurm.conf 設定

#### AccountingStorageTRES

用於定義系統上要追蹤哪些 TRES。預設追蹤 Billing、CPU、Energy、Memory、Node、FS/Disk、Pages 和 VMem。這些預設 TRES 無法停用，只能新增其他 TRES。

```
# 追蹤 GPU GRES 和 iop1 授權
AccountingStorageTRES=gres/gpu,license/iop1
```

這將追蹤 billing、cpu、energy、memory、nodes、fs/disk、pages 和 vmem，以及名為 gpu 的 GRES 和名為 iop1 的授權。

**需要關聯名稱的 TRES**：
- BB（與突發緩衝外掛程式同名）
- GRES
- License

**GRES 子類型建議**：

包含特定 GRES 子類型時，建議也包含其通用類型，否則只請求通用類型的作業不會被計費：

```
# 同時追蹤通用 gpu 和特定 tesla 子類型
AccountingStorageTRES=gres/gpu,gres/gpu:tesla
```

**注意**：設定 gres/gpu 也會自動設定 gres/gpumem 和 gres/gpuutil。

---

#### PriorityWeightTRES

以逗號分隔的 TRES 類型和權重清單，設定每種 TRES 類型對作業優先順序的貢獻程度：

```
PriorityWeightTRES=CPU=1000,Mem=2000,GRES/gpu=3000
```

**適用條件**：
- `PriorityType=priority/multifactor`
- 每種 TRES 類型都已在 AccountingStorageTRES 中設定

**注意**：Billing TRES 無法用於優先順序計算，因為數值是在作業分配資源後才產生（不同分割區數值可能不同）。

---

#### TRESBillingWeights

以逗號分隔的 `<TRES 類型>=<數值權重>` 配對清單，定義一個或多個追蹤 TRES 類型的計費權重，用於計算每個分割區中作業的使用量。

```
TRESBillingWeights="CPU=1.0,Mem=0.25G,GRES/gpu=2.0,license/licA=1.5"
```

**用途**：
- 計算[公平分享](priority_multifactor.md)
- 強制針對 billing TRES 設定的[資源限制](resource_limits.md)

**預設設定**：
```
TRESBillingWeights="CPU=1"
```

**單位調整**：

資源的加權數量可透過在計費權重後新增 `[KMGTP]` 後綴來調整。記憶體和突發緩衝的基本單位是 MB。

| 設定 | 說明 |
|------|------|
| `mem=0.25` | 每 MB 0.25 單位 |
| `mem=0.25G` | 每 GB 0.25 單位 |

**範例**：8GB 記憶體
- `mem=0.25`：8192MB × 0.25 = 2048 計費單位
- `mem=0.25G`：(8192MB/1024) × 0.25 = 2 計費單位

---

### 計費計算

#### 預設計算（加總）

預設 TRES 計費計算為所有 TRES 類型乘以對應計費權重的總和。

**範例**：
```
TRESBillingWeights="CPU=1.0,Mem=0.25G,GRES/gpu=2.0,license/licA=1.5"
```

分配 1 CPU 和 8GB 記憶體的作業：
```
(1×1.0) + (8×0.25) + (0×2.0) + (0×1.5) = 3.0
```

**注意**：TRES 權重可以是浮點數，但最終計費金額會截斷為整數。

#### MAX_TRES 計算

設定 `PriorityFlags=MAX_TRES` 時，可計費 TRES 計算為：
- 個別 TRES（綁定到特定節點，如 cpus、mem、gres）的**最大值**
- 全域 TRES（任何節點上可用，如 licenses）的**總和**

使用上述範例：
```
MAX(1×1.0, 8×0.25, 0×2.0) + (0×1.5) = 2.0
```

#### MAX_TRES_GRES 計算

設定 `PriorityFlags=MAX_TRES_GRES` 時，可計費 TRES 計算為：
- 所有可計費 GRES 的**總和**
- 其他個別 TRES 的**最大值**
- 全域 TRES 的**總和**

使用上述範例（無 GPU）：
```
(0×2.0) + MAX(1×1.0, 8×0.25) + (0×1.5) = 2.0
```

新增 1 個 GPU：
```
(1×2.0) + MAX(1×1.0, 8×0.25) + (0×1.5) = 4.0
```

---

### sacct

使用 sacct 查看每個作業的 TRES：

```bash
sacct --format=jobid,tres,elapsed
```

---

### sacctmgr

使用 sacctmgr 查看系統中全域可用的各種 TRES：

```bash
sacctmgr show tres
```

---

### sreport

sreport 報告不同的 TRES。使用逗號分隔的 `--tres=` 選項可讓 sreport 產生請求 TRES 類型的報表。

```bash
sreport cluster utilization --tres=cpu,mem,gres/gpu
```

**Billing TRES 報告計算**：

在 sreport 中，報告的 Billing TRES 是從每個節點的最大 Billing TRES 乘以時間框架計算的。

- 如果節點屬於多個分割區且每個分割區定義了不同的 TRESBillingWeights，則節點的 Billing TRES 將是分割區中最高的
- 如果節點沒有任何分割區定義 TRESBillingWeights，則 Billing TRES 等於節點上的 CPU 數量

---

## 說明

### TRES 用途對照

| 用途 | 設定 | 說明 |
|------|------|------|
| 追蹤使用量 | AccountingStorageTRES | 記錄資源使用 |
| 計費權重 | TRESBillingWeights | 影響公平分享和限制 |
| 優先順序權重 | PriorityWeightTRES | 影響作業優先順序 |

### 計費模式比較

| 模式 | 設定 | 計算方式 |
|------|------|----------|
| 預設 | 無 | SUM(所有 TRES × 權重) |
| MAX_TRES | PriorityFlags=MAX_TRES | MAX(個別) + SUM(全域) |
| MAX_TRES_GRES | PriorityFlags=MAX_TRES_GRES | SUM(GRES) + MAX(其他個別) + SUM(全域) |

---

## 實務範例

### 基本 TRES 設定

```
# slurm.conf
AccountingStorageTRES=gres/gpu,license/matlab

# 分割區計費權重
PartitionName=gpu TRESBillingWeights="CPU=1.0,Mem=0.5G,GRES/gpu=5.0"
PartitionName=cpu TRESBillingWeights="CPU=1.0,Mem=0.5G"
```

### GPU 叢集計費設定

```
# slurm.conf
AccountingStorageTRES=gres/gpu,gres/gpu:a100,gres/gpu:v100

# 不同 GPU 有不同計費權重
PartitionName=a100 TRESBillingWeights="CPU=1.0,GRES/gpu:a100=10.0"
PartitionName=v100 TRESBillingWeights="CPU=1.0,GRES/gpu:v100=5.0"
```

### 優先順序權重設定

```
# slurm.conf
PriorityType=priority/multifactor
PriorityWeightTRES=CPU=1000,Mem=500,GRES/gpu=2000
```

### 查詢 TRES 資訊

```bash
# 查看系統 TRES
sacctmgr show tres

# 查看作業 TRES 使用
sacct -j 12345 --format=jobid,tres,elapsed

# 產生 TRES 報表
sreport cluster utilization --tres=cpu,mem,gres/gpu start=2024-01-01
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 只設定 GRES 子類型 | 同時設定通用類型和子類型 |
| 忽略單位後綴 | 記憶體使用 G 後綴更直觀 |
| 計費權重過於複雜 | 保持簡單，易於理解 |
| 未考慮 MAX_TRES 影響 | 根據站點需求選擇計費模式 |

### 建議

1. **規劃 TRES 追蹤**：
   - 只追蹤需要的 TRES
   - 考慮報表和限制需求

2. **計費權重設計**：
   - 反映資源的實際成本
   - 保持一致性和可預測性

3. **定期檢查**：
   ```bash
   # 檢查 TRES 使用情況
   sreport cluster utilization --tres=ALL
   ```

---

## 快速參考

### slurm.conf 設定

```
# 追蹤 TRES
AccountingStorageTRES=gres/gpu,license/matlab

# 計費權重
TRESBillingWeights="CPU=1.0,Mem=0.5G,GRES/gpu=5.0"

# 優先順序權重
PriorityWeightTRES=CPU=1000,Mem=500,GRES/gpu=2000

# 計費模式
# PriorityFlags=MAX_TRES
# PriorityFlags=MAX_TRES_GRES
```

### 常用命令

| 命令 | 功能 |
|------|------|
| `sacctmgr show tres` | 顯示系統 TRES |
| `sacct --format=tres` | 顯示作業 TRES |
| `sreport cluster utilization --tres=X` | TRES 利用率報表 |

### 預設追蹤的 TRES

- Billing
- CPU
- Energy
- Memory
- Node
- FS/Disk
- Pages
- VMem

### 相關文件

- [計費](accounting.md) - 計費系統設定
- [GRES](gres.md) - 通用資源設定
- [優先順序](priority_multifactor.md) - 多因子優先順序
- [資源限制](resource_limits.md) - 資源限制設定
