---
title: Slurm REST API Account 限制與強制執行機制分析
description: 分析 Slurm REST API 如何更新 Account 層級限制，以及 AccountingStorageEnforce 參數對限制生效的決定性影響
author: Paige (Technical Writer Agent)
date: 2026-01-29
---

# Slurm REST API Account 限制與強制執行機制分析

本文檔深入探討透過 Slurm REST API 修改 Association 限制時的行為，特別是針對 Account 層級的設定，以及這些設定如何依賴 `slurm.conf` 中的 `AccountingStorageEnforce` 參數才能真正生效。

## 1. 發現概述

在分析 Slurm 程式碼與 API 行為後，我們確認了兩個關鍵機制：

1.  **API 對 Account 限制的修改**：Payload 中指定 `user: ""` (空字串) 會直接修改 Account 本身的 Association，而非特定使用者。若將 `max_jobs` 設為 `0`，會導致該 Account 下**所有**使用者因階層限制而被阻擋。
2.  **強制執行依賴性**：即便資料庫中的限制值已被更新，Slurm Controller (`slurmctld`) **只有**在設定了 `AccountingStorageEnforce=limits` (或包含 `limits`) 時，才會執行這些檢查。否則，這些限制形同虛設。

---

## 2. 技術細節分析

### 2.1 API 行為：Account Association 的鎖定機制

當透過 `POST /slurmdb/v0.0.44/associations/` 發送如下 Payload 時：

```json
{
  "associations": [{
    "account": "project-alpha",
    "cluster": "mycluster",
    "user": "",
    "max": {
      "jobs": {
        "per": { "count": 0, "submitted": 0 }
      }
    }
  }]
}
```

**程式碼路徑分析** (`src/slurmrestd/plugins/openapi/slurmdbd/associations.c`)：

- 處理函式 `_foreach_update_assoc` 會接收到 `user` 為空字串。
- 在 Slurm Database (SlurmDBD) 的定義中，`user=""` 且 `account="project-alpha"` 代表這是 **Account Association**。
- `0` 在 Slurm 中明確代表數值 **零** (Zero)，而無限制通常由 `-1` (`INFINITE`) 表示。

**影響結果**：
Slurm 採用階層式限制 (Hierarchical Limits)。Account 是其下所有 User 的父節點。
- 當 Account 的 `MaxJobs` 被設為 0，該 Account 擁有的總作業額度即為 0。
- 該 Account 下的**所有使用者** (User A, User B...) 都共享此額度。
- 結果：所有使用者都無法提交或執行作業，該 Account 形同被「凍結」。

### 2.2 強制執行開關：AccountingStorageEnforce

這是最關鍵的發現。僅修改資料庫並不足以阻止作業執行。

**程式碼邏輯** (`src/slurmctld/acct_policy.c`):

在函式 `_adjust_limit_usage` 中，有如下檢查：

```c
if (!(accounting_enforce & ACCOUNTING_ENFORCE_LIMITS)
    || !_valid_job_assoc(job_ptr))
    return;
```

- **變數來源**：`accounting_enforce` 來自 `slurm.conf` 的 `AccountingStorageEnforce` 設定。
- **邏輯判斷**：如果設定中不包含 `limits` 旗標 (`ACCOUNTING_ENFORCE_LIMITS`)，函式直接返回 (`return`)。
- **後果**：`slurmctld` 完全跳過對資料庫中限制值的檢查與計算。作業將被允許提交和執行，無視 DB 中的 `MaxJobs=0` 設定。

---

## 3. 系統流程圖

以下 Mermaid 序列圖展示了從 API 呼叫到作業提交的完整決策流程：

```mermaid
sequenceDiagram
    participant Admin as 管理員 (API Client)
    participant REST as slurmrestd
    participant DB as SlurmDBD (Database)
    participant User as 使用者 (sbatch)
    participant Ctl as slurmctld (Controller)
    participant Conf as slurm.conf

    Note over Admin, DB: 階段 1: 設定限制 (API)
    Admin->>REST: POST /associations (user="", max_jobs=0)
    REST->>DB: Update Account Association (project-alpha)
    DB-->>REST: Success
    REST-->>Admin: 200 OK
    Note right of DB: DB 中 project-alpha 的 MaxJobs 現為 0

    Note over User, Conf: 階段 2: 提交作業 (強制執行檢查)
    User->>Ctl: sbatch (account=project-alpha)
    Ctl->>Conf: 讀取 AccountingStorageEnforce
    
    alt AccountingStorageEnforce 包含 "limits"
        Conf-->>Ctl: ENFORCE_LIMITS = True
        Ctl->>DB: 查詢 project-alpha 限制
        DB-->>Ctl: MaxJobs=0
        Ctl->>Ctl: 檢查: 0 (Limit) vs 1 (Request)
        Ctl-->>User: 拒絕提交 (Job violates accounting policy)
    else AccountingStorageEnforce 未設定或無 "limits"
        Conf-->>Ctl: ENFORCE_LIMITS = False
        Ctl->>Ctl: 跳過限制檢查 (_adjust_limit_usage returns)
        Ctl-->>User: 接受提交 (JobID: 12345)
        Note right of Ctl: 限制無效化
    end
```

---

## 4. AccountingStorageEnforce 選項全解析

`AccountingStorageEnforce` 是控制 Slurm 是否以及如何執行會計策略的總開關。

### 4.1 選項層級與隱含行為

在 `src/common/read_config.c` 的 `_validate_accounting_storage_enforce` 函式中，定義了各選項的層級關係。許多選項會**自動開啟**依賴的較低層級選項。

| 選項 | 隱含開啟的選項 | 功能描述 |
| :--- | :--- | :--- |
| **associations** | 無 | 強制執行關聯性檢查。使用者提交作業時，必須在 DB 中有對應的 Association (User/Account/Cluster/Partition 組合)，否則拒絕提交。 |
| **limits** | **associations** | 強制執行定義在 Association 或 QOS 上的資源限制 (如 `MaxJobs`, `MaxTRES` 等)。 |
| **safe** | **associations**, **limits** | 啟用「安全」的限制檢查模式。若作業**預計**會導致超額 (如時間積分耗盡)，則拒絕啟動作業。 |
| **qos** | **associations** | 強制執行 QOS (Quality of Service) 相關規則。 |
| **wckeys** | **associations** | 強制執行 WCKey (Workload Characterization Key) 檢查。 |
| **nojobs** | **nosteps** | 不將作業資訊寫入會計資料庫 (僅用於測試或特殊用途)。 |
| **nosteps** | 無 | 不將作業步驟 (Step) 資訊寫入會計資料庫。 |
| **all** | **associations**, **limits**, **safe**, **qos**, **wckeys** | 啟用所有強制執行選項 (但不包含 `nojobs` 和 `nosteps`)。 |

### 4.2 深入解析：Safe 模式

在 `limits` 模式下，Slurm 僅檢查「當前已用」資源是否超標。這可能導致作業跑到一半因額度耗盡而被殺死。`safe` 模式則引入了預測機制。

#### 機制比較

1.  **無 Safe (預設)**：
    - 檢查：`CurrentUsage < Limit`
    - 風險：作業執行中途 `CurrentUsage` 增加導致 `CurrentUsage > Limit`，作業被強制終止 (Kill)。

2.  **有 Safe**：
    - 檢查：`CurrentUsage + (JobTimeLimit * JobResources) < Limit`
    - 優勢：Slurm 會計算作業跑完所需的總消耗。若預測會超標，作業將保持 `Pending` 狀態而不予啟動，避免浪費資源。

#### 決策流程圖

```mermaid
flowchart TD
    Submit[使用者提交作業<br>預計執行 60 分鐘] --> CheckCurrent{檢查當前額度<br>是否已滿?}
    
    CheckCurrent -->|已滿| Deny[拒絕/排隊]
    CheckCurrent -->|未滿| IsSafeSet{是否設定了 safe?}
    
    IsSafeSet -->|No (預設)| RunJob[立即執行作業]
    RunJob -->|執行中...| LimitHit{執行中途<br>額度耗盡?}
    LimitHit -->|是| KillJob[☠️ 作業被強制殺死<br>浪費運算資源]
    LimitHit -->|否| Finish[作業完成]

    IsSafeSet -->|Yes (safe)| Predict{預測:<br>當前 + 預計使用量 > 限額?}
    Predict -->|會超標| PendJob[✋ 暫緩執行 (Pending)<br>等待額度足夠]
    Predict -->|不會超標| SafeRun[安全執行作業]
    SafeRun --> FinishSafe[作業順利完成<br>不會因額度被殺]

    style KillJob fill:#ffcccc,stroke:#ff0000
    style SafeRun fill:#ccffcc,stroke:#00aa00
    style PendJob fill:#ffffcc,stroke:#aaaa00
```

---

## 5. 設定建議

為了確保 API 設定的限制能夠生效，必須在 `slurm.conf` 中正確配置。

### 推薦設定

```bash
# slurm.conf
AccountingStorageType=accounting_storage/slurmdbd
# 關鍵設定：必須包含 limits
AccountingStorageEnforce=associations,limits,qos
```

### 設定選項說明

| 選項 | 說明 | 對 API 限制的影響 |
| :--- | :--- | :--- |
| **(未設定)** | 預設行為 (none) | **限制無效**。Controller 忽略 DB 中的限制值。 |
| **limits** | 強制執行資源與作業計數限制 | **限制生效**。MaxJobs=0 會阻止作業。 |
| **associations** | 強制執行關聯性 (User 必須在 DB 中) | 僅檢查身分，不檢查數值限制。 |
| **safe** | 若作業會超過限制則不予啟動 | 增強限制執行的安全性。 |

## 5. 結論

要透過 REST API 成功凍結一個 Account，必須同時滿足兩個條件：
1.  **資料層面**：API 呼叫正確將 Account Association 的 `MaxJobs` 設為 `0`。
2.  **執行層面**：`slurm.conf` 中必須啟用 `AccountingStorageEnforce=limits`。

若缺其一，限制將不會產生預期的阻擋效果。

## 6. 相關文件參考

建議閱讀以下文件以獲得更全面的背景知識：

-   [Slurm 帳務與資源限制](../../doc/html/zh-tw/administrators/accounting.md) - 帳務系統基礎架構與配置概覽。
-   [資源限制設定](../../doc/html/zh-tw/administrators/resource_limits.md) - 詳細的資源限制管理說明。

