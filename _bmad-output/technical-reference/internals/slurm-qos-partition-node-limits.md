---
title: Slurm 資源限制強制機制深度解析
description: 深入分析 Slurm 排程器中 Partition 限制、QOS TRES 限制與三個控制維度（AccountingStorageEnforce / EnforcePartLimits / DenyOnLimit）的交互機制
author: Paige (BMAD Tech Writer)
date: 2026-03-06
---

# Slurm 資源限制強制機制深度解析

## 概述

Slurm 中「限制」和「限制是否被強制執行」是兩件不同的事。管理員可以在 Partition 和 QOS 上設定各種限制，但這些限制是否在提交時拒絕作業、還是在排程時 PENDING hold、還是完全不檢查 — 取決於三個獨立的控制維度。

本文基於 Slurm 25.11 原始碼，解析這三個維度如何交互。

---

## 三個控制維度

Slurm 的限制強制由三個**獨立**的設定控制，各自影響不同類型的限制：

```mermaid
flowchart TD
    subgraph layer1["AccountingStorageEnforce（slurm.conf 全域）"]
        ASE{"包含 limits?"}
        ASE -->|否| SKIP["所有 QOS/Association\nTRES 限制被跳過\n提交時+排程時都不檢查"]
        ASE -->|是| LAYER2["TRES 限制系統啟用"]
    end

    subgraph layer2["DenyOnLimit（QOS flag，每個 QOS 獨立）"]
        DOL{"QOS 有 DenyOnLimit?"}
        DOL -->|否| ACCEPT["提交通過\n排程時 PENDING hold"]
        DOL -->|是| STRICT["提交時嚴格檢查\n超限 → 拒絕提交"]
    end

    subgraph layer3["EnforcePartLimits（slurm.conf 全域）"]
        EPL{"設定值？"}
        EPL -->|"NO（預設）"| PART_SKIP["Partition 限制\n不在提交時拒絕"]
        EPL -->|"YES=ANY / ALL"| PART_CHECK["Partition 限制\n在提交時拒絕"]
    end

    LAYER2 --> DOL
    ASE -.->|"獨立運作"| EPL
```

| 控制項 | 設定位置 | 影響範圍 | 預設值 |
|---|---|---|---|
| **`AccountingStorageEnforce`** | `slurm.conf` | QOS/Association 的所有 TRES 限制（總開關） | `none` |
| **`DenyOnLimit`** | QOS flag | 提交時是否拒絕超過 TRES 限制的作業 | 未設定 |
| **`EnforcePartLimits`** | `slurm.conf` | Partition 的 MaxNodes/MinNodes/MaxTime | `NO` |

### 完整設定矩陣

| `AccountingStorageEnforce` | `EnforcePartLimits` | `DenyOnLimit` | 提交時 Partition 限制 | 提交時 TRES 限制 | 排程時 TRES 限制 |
|---|---|---|---|---|---|
| `none` | `NO` | 無 | 不檢查 | 不檢查 | **不檢查** |
| `limits` | `NO` | 無 | 不檢查 | 不檢查 | **檢查** |
| `limits` | `NO` | 有 | 不檢查 | **拒絕** | N/A |
| `limits` | `ALL` | 無 | **拒絕** | 不檢查 | **檢查** |
| `limits` | `ALL` | 有 | **拒絕** | **拒絕** | N/A |
| `none` | `ALL` | 有 | **拒絕** | **不檢查** | **不檢查** |

> **關鍵**：最後一行說明 `AccountingStorageEnforce` 不包含 `limits` 時，即使設了 `DenyOnLimit`，TRES 限制也不會被檢查。`DenyOnLimit` 只控制內部的 `strict_checking`，但外層的 `ACCOUNTING_ENFORCE_LIMITS` 閘門更早就把整個呼叫擋掉了。

---

## `AccountingStorageEnforce` — 會計系統的總開關

**原始碼位置**：`src/common/read_config.c:3420-3479`、`src/common/read_config.h:66-73`

### 設定值解析

這是一個 bitmask 設定，支援以逗號分隔的多個值：

```c
#define ACCOUNTING_ENFORCE_ASSOCS  SLURM_BIT(0)  // associations
#define ACCOUNTING_ENFORCE_LIMITS  SLURM_BIT(1)  // limits
#define ACCOUNTING_ENFORCE_WCKEYS  SLURM_BIT(2)  // wckeys
#define ACCOUNTING_ENFORCE_QOS     SLURM_BIT(3)  // qos
#define ACCOUNTING_ENFORCE_SAFE    SLURM_BIT(4)  // safe
#define ACCOUNTING_ENFORCE_NO_JOBS SLURM_BIT(5)  // nojobs
#define ACCOUNTING_ENFORCE_NO_STEPS SLURM_BIT(6) // nosteps
#define ACCOUNTING_ENFORCE_TRES    SLURM_BIT(7)  // tres
```

解析邏輯中，高級值會自動包含低級值：

| slurm.conf 設定值 | 自動啟用的 flags |
|---|---|
| `associations` | `ASSOCS` |
| `limits` | `ASSOCS` + `LIMITS` |
| `safe` | `ASSOCS` + `LIMITS` + `SAFE` |
| `qos` | `ASSOCS` + `QOS` |
| `wckeys` | `ASSOCS` + `WCKEYS` |
| `all` | 所有 flags（但不含 `nojobs`/`nosteps`） |

### `ACCOUNTING_ENFORCE_LIMITS` 的閘門效應

**`ACCOUNTING_ENFORCE_LIMITS` 是所有 QOS/Association TRES 限制檢查的硬性前提。** 沒有這個 flag，整條檢查鏈被完全跳過 — 提交時和排程時都不會檢查任何 TRES 限制。

閘門位置列表：

| 檢查點 | 原始碼位置 | 效果 |
|---|---|---|
| 提交時 | `job_mgr.c:7470` | 跳過 `acct_policy_validate()` |
| 排程 pre-select | `acct_policy.c:3815` | 直接 `return true` |
| 排程 post-select | `acct_policy.c:4059` | 直接 `return true` |
| max_nodes 計算 | `acct_policy.c:4417` | 回傳無限制值 |

### `ACCOUNTING_ENFORCE_SAFE` — 安全限制模式

`safe` 影響排程時的使用量檢查。在 `_validate_tres_usage_limits()` 中（第 1593 行），`safe_limits` 控制是否檢查「請求量 + 已使用量」是否超過限制。

但注意：`MaxTRESPerUser` 的檢查在 `_qos_job_runnable_post_select()` 中被**硬編碼**為 `safe_limits=true`（第 2690 行），不受 `ACCOUNTING_ENFORCE_SAFE` 影響。`ACCOUNTING_ENFORCE_SAFE` 主要影響 `GrpTRES` 和 `GrpTRESRunMins` 等群組使用量的檢查。

---

## `EnforcePartLimits` — Partition 限制的強制設定

**原始碼位置**：`src/common/slurm_protocol_defs.c:5661-5686`

### 設定值解析

```c
if (!xstrcasecmp(value, "yes")
    || !xstrcasecmp(value, "up")
    || !xstrcasecmp(value, "true")
    || !xstrcasecmp(value, "1") || !xstrcasecmp(value, "any")) {
    *param = PARTITION_ENFORCE_ANY;      // YES = ANY
} else if (!xstrcasecmp(value, "no")
           || !xstrcasecmp(value, "down")
           || !xstrcasecmp(value, "false")
           || !xstrcasecmp(value, "0")) {
    *param = PARTITION_ENFORCE_NONE;     // NO
} else if (!xstrcasecmp(value, "all")) {
    *param = PARTITION_ENFORCE_ALL;      // ALL
}
```

| slurm.conf 設定值 | 內部常數 | 含義 |
|---|---|---|
| `NO` / `false` / `0` / `down` | `PARTITION_ENFORCE_NONE` (0) | 不強制（預設） |
| **`YES`** / `true` / `1` / `up` / `any` | **`PARTITION_ENFORCE_ANY`** | 至少一個 partition 通過即可 |
| `ALL` | `PARTITION_ENFORCE_ALL` | 所有 partition 都必須通過 |

> **注意**：`YES` 等同 `ANY`，**不是** `ALL`。這是 Slurm 的設計行為。

### ANY vs ALL：單一 vs 多 Partition 行為差異

**原始碼位置**：`src/slurmctld/job_mgr.c:6699-6813`

**單一 Partition 時**，ANY 和 ALL 行為完全一致 — 走 `_valid_job_part()` 的 `else` 分支（第 6799-6813 行），判斷條件 `slurm_conf.enforce_part_limits` 非零即為 true。

**多 Partition 時**（如 `sbatch -p partA,partB`），走 `_foreach_valid_part()` 迴圈（第 6699-6745 行），行為有本質差異：

```c
// _foreach_valid_part() — 對每個 partition 執行
rc = _part_access_check(part_ptr, ...);

if ((rc != SLURM_SUCCESS) &&
    ((rc == ESLURM_ACCESS_DENIED) ||
     (rc == ESLURM_USER_ID_MISSING) ||
     (slurm_conf.enforce_part_limits == PARTITION_ENFORCE_ALL))) {
    // ALL 模式：任何一個 partition 失敗 → 立即中止迴圈，拒絕提交
    foreach_valid_part->rc = rc;
    return -1;
} else if (rc != SLURM_SUCCESS) {
    // ANY 模式：記錄錯誤但繼續檢查下一個 partition
    foreach_valid_part->rc = rc;
} else {
    // 此 partition 通過
    foreach_valid_part->any_check = true;
}

// ANY 模式：只要找到一個通過的 partition，整體就成功
if (foreach_valid_part->any_check &&
    (slurm_conf.enforce_part_limits == PARTITION_ENFORCE_ANY))
    foreach_valid_part->rc = SLURM_SUCCESS;
```

| 場景 | `NO` | `ANY`（= `YES`） | `ALL` |
|---|---|---|---|
| **單一 partition，檢查失敗** | 忽略，進入佇列 | 拒絕提交 | 拒絕提交 |
| **多 partition，全部失敗** | 忽略，進入佇列 | 拒絕提交 | 拒絕提交 |
| **多 partition，部分失敗** | 忽略，進入佇列 | 接受（至少一個通過） | 拒絕提交（第一個失敗即中止） |
| **多 partition，全部通過** | 接受 | 接受 | 接受 |

```mermaid
flowchart TD
    subgraph single_part["單一 Partition"]
        SP_CHECK{"partition\n檢查結果？"}
        SP_CHECK -->|通過| SP_OK["接受"]
        SP_CHECK -->|失敗| SP_EPL{"EnforcePartLimits?"}
        SP_EPL -->|NO| SP_ACCEPT["忽略，進入佇列"]
        SP_EPL -->|"ANY 或 ALL\n（行為相同）"| SP_REJECT["拒絕提交"]
    end

    subgraph multi_part["多 Partition（如 partA,partB）"]
        MP_LOOP["逐一檢查每個 partition"]
        MP_LOOP --> MP_EPL{"EnforcePartLimits?"}

        MP_EPL -->|ALL| MP_ALL{"任一 partition\n失敗？"}
        MP_ALL -->|是| MP_ALL_REJECT["立即中止\n拒絕提交"]
        MP_ALL -->|"全部通過"| MP_ALL_OK["接受"]

        MP_EPL -->|ANY| MP_ANY{"至少一個\npartition 通過？"}
        MP_ANY -->|是| MP_ANY_OK["接受\n（忽略失敗的 partition）"]
        MP_ANY -->|"全部失敗"| MP_ANY_REJECT["拒絕提交"]
    end
```

多 partition 情境下會聚合所有 partition 的限制 — 取最小的 min_nodes、最大的 max_nodes 和 max_time（第 6736-6743 行），讓後續驗證使用最寬鬆的限制。

---

## Partition 節點限制的檢查流程

### 第一階段：提交驗證

**`_qos_part_check()`**（`src/slurmctld/job_mgr.c:6395-6443`）負責比較作業請求與 Partition 的節點限制：

```c
if ((part_ptr->state_up & PARTITION_SCHED) &&
    (qos_part_check->max_nodes != NO_VAL) &&
    (qos_part_check->max_nodes > part_ptr->max_nodes) &&
    (!qos_ptr || !(qos_ptr->flags & QOS_FLAG_PART_MAX_NODE))) {
    qos_part_check->error_code = ESLURM_INVALID_NODE_COUNT;
    return -1;
}
```

但 `_qos_part_check()` 設定的錯誤碼**並不一定導致作業被拒絕**。外層的 **`_valid_job_part()`**（第 6748-6813 行）根據 `EnforcePartLimits` 決定最終行為：

```c
rc = _part_access_check(part_ptr, ...);
if ((rc != SLURM_SUCCESS) &&
    ((rc == ESLURM_ACCESS_DENIED) ||
     (rc == ESLURM_USER_ID_MISSING) ||
     slurm_conf.enforce_part_limits))   // ← 關鍵：預設為 0，跳過
    goto fini;
/* Enforce Part Limit = no */
rc = SLURM_SUCCESS;                     // ← 預設：忽略錯誤
```

### 第二階段：排程器持續檢查

**原始碼位置**：`src/slurmctld/job_mgr.c:6953-7030`

排程器在每個排程週期透過 `_job_runnable_test2()` → `job_limits_check()` 持續驗證已入佇列的作業。此檢查**不受 `EnforcePartLimits` 影響**，直接將結果轉為 pending reason：

```c
if ((rc = _part_access_check(part_ptr, &job_desc, ...))) {
    switch (rc) {
    case ESLURM_INVALID_NODE_COUNT:
        fail_reason = WAIT_PART_NODE_LIMIT;   // → PartitionNodeLimit
        break;
    case ESLURM_INVALID_TIME_LIMIT:
        if (job_ptr->limit_set.time != ADMIN_SET_LIMIT)
            fail_reason = WAIT_PART_TIME_LIMIT;
        break;
    default:
        fail_reason = WAIT_PART_CONFIG;
        break;
    }
}
```

| 特性 | 提交階段 | 排程器檢查階段 |
|---|---|---|
| 呼叫函式 | `_part_access_check()` | `_part_access_check()`（相同） |
| 受 `EnforcePartLimits` 影響 | **是** | **否** |
| 失敗結果 | 拒絕提交或忽略 | 設定 pending reason |
| 執行時機 | 一次（提交時） | 每個排程週期持續檢查 |
| 條件改善後 | N/A | 自動清除 reason，允許排程 |

### 第三階段：排程決策 (`get_node_cnts`)

**原始碼位置**：`src/slurmctld/node_scheduler.c:3144-3203`

這是決定作業**實際能獲得多少節點**的核心函式：

```c
// === 步驟 1：決定 min_nodes ===
if (qos_flags & QOS_FLAG_PART_MIN_NODE)
    *min_nodes = job_ptr->details->min_nodes;       // 覆蓋：用 Job 請求值
else
    *min_nodes = MAX(job_min, part_min);             // 預設：取較大值

// === 步驟 2：決定 max_nodes（三條路徑）===
if (!job_ptr->details->max_nodes)
    *max_nodes = part_ptr->max_nodes;                // Case A: Job 沒指定
else if (qos_flags & QOS_FLAG_PART_MAX_NODE)
    *max_nodes = job_ptr->details->max_nodes;        // Case B: QOS 覆蓋
else
    *max_nodes = MIN(job_max, part_max);             // Case C: 預設取嚴格

// === 步驟 3：會計限制永遠生效 ===
acct_max_nodes = acct_policy_get_max_nodes(job_ptr, &wait_reason);
*max_nodes = MIN(*max_nodes, acct_max_nodes);        // 再取 MIN
```

```mermaid
flowchart TD
    START["get_node_cnts()"] --> CHECK_JOB{"Job 有指定\nmax_nodes?"}

    CHECK_JOB -->|"沒有\n(max_nodes=0)"| CASE_A["Case A\nmax_nodes = Partition MaxNodes"]

    CHECK_JOB -->|有| CHECK_FLAG{"QOS 有\nPartitionMaxNodes\nflag?"}

    CHECK_FLAG -->|有| CASE_B["Case B（覆蓋模式）\nmax_nodes = Job 請求值\n跳過 Partition 限制"]

    CHECK_FLAG -->|沒有| CASE_C["Case C（預設模式）\nmax_nodes = MIN(Job請求, Partition)"]

    CASE_A --> ACCT["會計限制\nacct_policy_get_max_nodes()"]
    CASE_B --> ACCT
    CASE_C --> ACCT

    ACCT --> FINAL["最終 max_nodes =\nMIN(上一步結果, 會計限制)"]

    FINAL --> VALID{"max_nodes >= min_nodes?"}
    VALID -->|是| OK["排程成功"]
    VALID -->|否| FAIL["ESLURM_REQUESTED_PART_CONFIG_UNAVAILABLE"]
```

| 模式 | 觸發條件 | max_nodes 計算 | 設計目的 |
|---|---|---|---|
| **預設模式** | QOS 未設 `PartitionMaxNodes` | `MIN(Job, Partition)` | 安全優先，防止超配 |
| **覆蓋模式** | QOS 已設 `PartitionMaxNodes` | Job 請求值（無視 Partition） | 允許特權 QOS 突破 Partition 限制 |

關鍵設計：**無論哪種模式，會計限制（步驟 3）永遠生效**。覆蓋的是 Partition 限制，不是 QOS 自身的資源配額。

---

## QOS TRES 限制的檢查流程

前提：`AccountingStorageEnforce` 必須包含 `limits`，否則以下所有檢查都被跳過。

### 提交時：`DenyOnLimit` 決定是否拒絕

**原始碼位置**：`src/slurmctld/acct_policy.c:1685-1760`

提交時的呼叫鏈：`acct_policy_validate()` → `_qos_policy_validate()` → `_validate_tres_limits_for_qos()`

```c
// acct_policy.c:1351-1352 — _validate_tres_limits_for_qos()
if (!strict_checking)
    return true;    // ← 沒有 DenyOnLimit → 直接跳過所有 TRES 限制檢查
```

`strict_checking` 取決於 QOS 是否設定了 `DenyOnLimit`：

```c
// acct_policy.c:3290-3293 — _acct_policy_validate()
strict_checking = (qos_ptr_1->flags & QOS_FLAG_DENY_LIMIT);
if (qos_ptr_2 && !strict_checking)
    strict_checking = qos_ptr_2->flags & QOS_FLAG_DENY_LIMIT;
```

### 排程時：永遠檢查

**原始碼位置**：`src/slurmctld/acct_policy.c:2326-2730`

排程階段的 `_qos_job_runnable_post_select()` 使用 `_validate_tres_usage_limits_for_qos()` 檢查 TRES 使用量。`safe_limits` 參數被硬編碼為 `true`（第 2690 行），**不受 `DenyOnLimit` 影響**，永遠執行：

```c
// acct_policy.c:2686-2690 — 排程時的 MaxTRESPerUser 檢查
tres_usage = _validate_tres_usage_limits_for_qos(
    &tres_pos,
    qos_ptr->max_tres_pu_ctld, qos_out_ptr->max_tres_pu_ctld,
    tres_req_cnt, used_limits->tres,
    NULL, job_ptr->limit_set.tres, true);   // ← 硬編碼 safe_limits=true
```

### 行為對照

| 階段 | 沒有 `DenyOnLimit` | 有 `DenyOnLimit` |
|---|---|---|
| **提交時** | 跳過 TRES 檢查，作業進入佇列 | 嚴格檢查，超限則拒絕（`ESLURM_ACCOUNTING_POLICY`） |
| **排程時** | 檢查 TRES 使用量，超限則 PENDING hold | N/A（已在提交時被拒絕） |

---

## 會計限制層：`acct_policy_get_max_nodes`

**原始碼位置**：`src/slurmctld/acct_policy.c:4402-4518`

不管前面怎麼算，排程決策的最後一步永遠要過這一關（前提是 `ACCOUNTING_ENFORCE_LIMITS` 已啟用）。

```mermaid
flowchart TD
    subgraph qos_limits["QOS 限制層"]
        PA["MaxTRESPerAccount\n(每帳戶上限)"]
        PJ["MaxTRESPerJob\n(每作業上限)"]
        PU["MaxTRESPerUser\n(每使用者上限)"]
        GRP["GrpTRES\n(群組總量上限)"]
    end

    subgraph assoc_limits["Association 限制層"]
        A_GRP["GrpTRES\n(關聯群組上限)"]
        A_MAX["MaxTRES\n(關聯個別上限)"]
    end

    PA --> MIN_QOS["取最小值"]
    PJ --> MIN_QOS
    PU --> MIN_QOS
    MIN_QOS --> QOS_RESULT["qos_max_p_limit"]
    GRP --> MIN_ALL["取最小值"]
    QOS_RESULT --> MIN_ALL

    A_GRP --> MIN_ASSOC["取最小值\n（受 limit_factor 調整）"]
    A_MAX --> MIN_ASSOC
    MIN_ASSOC --> MIN_ALL

    MIN_ALL --> RESULT["最終 acct_max_nodes"]
```

### 雙 QOS 優先順序機制

Slurm 支援「Job QOS」與「Partition QOS」同時存在。`acct_policy_set_qos_order()` 決定誰是主要 QOS：

```c
if (job_ptr->qos_ptr->flags & QOS_FLAG_OVER_PART_QOS) {
    *qos_ptr_1 = job_ptr->qos_ptr;        // Job QOS 為主
    *qos_ptr_2 = job_ptr->part_ptr->qos_ptr; // Partition QOS 為輔
} else {
    *qos_ptr_1 = job_ptr->part_ptr->qos_ptr; // Partition QOS 為主（預設）
    *qos_ptr_2 = job_ptr->qos_ptr;           // Job QOS 為輔
}
```

**主要 QOS 的限制優先採用；輔助 QOS 只在主要 QOS 未設定某項限制時才填補。**

```mermaid
flowchart LR
    subgraph default_order["預設順序"]
        direction TB
        P1["1st: Partition QOS\n（主要）"]
        P2["2nd: Job QOS\n（填補未設定項）"]
    end

    subgraph over_part_qos["OverPartQOS 模式"]
        direction TB
        O1["1st: Job QOS\n（主要）"]
        O2["2nd: Partition QOS\n（填補未設定項）"]
    end

    default_order -.->|"Job QOS 設定\nOverPartQOS flag"| over_part_qos
```

---

## 情境對照表

假設環境配置：

- **Partition** `compute`：MinNodes=1, MaxNodes=5
- **Job QOS** `premium`：MaxTRESPerJob/Node 可變
- `AccountingStorageEnforce=limits`
- `EnforcePartLimits=NO`（預設值）
- Job 請求：`-N 10`

### 最大節點數情境

> 若設 `EnforcePartLimits=YES`（=`ANY`）或 `ALL`，情境 1 會在提交時被直接拒絕。

| 情境 | Partition Max | QOS MaxNodes/Job | QOS Flag | 提交結果 | 排程器狀態 | 排程 max_nodes |
|---|---|---|---|---|---|---|
| 1. 預設，QOS 較寬 | 5 | 8 | 無 | 接受 | PENDING (PartitionNodeLimit) | 無法排程 |
| 2. 預設，Job 請求合法 (`-N 4`) | 5 | 8 | 無 | 接受 | 可排程 | MIN(4, 5) = 4，再 MIN(4, 8) = **4** |
| 3. 預設，QOS 較嚴 (`-N 4`) | 5 | 3 | 無 | 接受 | 可排程 | MIN(4, 5) = 4，再 MIN(4, 3) = **3** |
| 4. 覆蓋，QOS 較寬 | 5 | 8 | `PartitionMaxNodes` | 接受 | 可排程 | Job=10，再 MIN(10, 8) = **8** |
| 5. 覆蓋，QOS 較嚴 | 5 | 3 | `PartitionMaxNodes` | 接受 | 可排程 | Job=10，再 MIN(10, 3) = **3** |
| 6. 覆蓋，QOS 無限制 | 5 | 無限制 | `PartitionMaxNodes` | 接受 | 可排程 | Job=10，再 MIN(10, INF) = **10** |

### 最小節點數情境

| 情境 | Partition Min | Job 請求 min | QOS Flag | 排程 min_nodes |
|---|---|---|---|---|
| 預設 | 3 | 1 | 無 | MAX(1, 3) = **3** |
| 覆蓋 | 3 | 1 | `PartitionMinNodes` | **1**（允許低於 Partition 下限） |

---

## sacctmgr 設定範例

### 基本 QOS 設定

```bash
# 建立 QOS，限制每作業最多 8 節點
sacctmgr add qos standard MaxTRESPerJob=node=8

# 建立特權 QOS — 覆蓋 Partition 限制
sacctmgr add qos premium \
    MaxTRESPerJob=node=20 \
    flags=PartitionMaxNodes

# 同時覆蓋最大與最小節點限制
sacctmgr add qos flexible \
    MaxTRESPerJob=node=50 \
    flags=PartitionMaxNodes,PartitionMinNodes

# 設定 OverPartQOS 改變 QOS 優先順序
sacctmgr add qos team_limited \
    MaxTRESPerJob=node=10 \
    flags=OverPartQOS
```

### DenyOnLimit 設定

```bash
# 提交時不拒絕（預設行為，排程時 PENDING hold）
sacctmgr add qos normal MaxTRESPerUser=cpu=2
# 結果：提交 -n 8 的作業會被接受，排程時 PENDING hold

# 提交時拒絕
sacctmgr add qos strict MaxTRESPerUser=cpu=2 flags=DenyOnLimit
# 結果：提交 -n 8 的作業會被立即拒絕
```

### 查看 QOS 設定

```bash
sacctmgr show qos format=name,flags,MaxTRESPerJob,MaxTRESPerUser
```

---

## 使用案例

### GPU 叢集的分層存取

```bash
# Partition 設定：PartitionName=gpu Nodes=gpu[01-04] MaxNodes=4

# 一般使用者 QOS
sacctmgr add qos gpu_normal MaxTRESPerJob=node=2

# 特權研究 QOS — 允許突破 Partition 限制
sacctmgr add qos gpu_research \
    MaxTRESPerJob=node=8 \
    flags=PartitionMaxNodes
```

- `gpu_normal` 使用者：最多 MIN(請求, 4, 2) = **2 節點**
- `gpu_research` 使用者：最多 MIN(請求, 8) = **最多 8 節點**（可超過 Partition 的 4）

### 教學環境的最小資源保證

```bash
# Partition 設定：PartitionName=teaching MinNodes=4

# 學生練習 QOS — 允許低於 Partition 最小值
sacctmgr add qos student_practice flags=PartitionMinNodes

# 學生作業 QOS — 遵循 Partition 最小值
sacctmgr add qos student_homework
```

- `student_practice`：`sbatch -N 1` 可以提交，min_nodes = 1
- `student_homework`：`sbatch -N 1` 實際 min_nodes = MAX(1, 4) = 4

---

## 設計哲學

```mermaid
flowchart LR
    subgraph limit_layers["限制層級（由外到內）"]
        direction TB
        L1["最外層：Partition 物理限制\n（可被 QOS flag 覆蓋）"]
        L2["中間層：QOS 資源配額\n（永遠生效）"]
        L3["最內層：Association 帳戶限制\n（永遠生效）"]
    end
```

| 設計原則 | 說明 |
|---|---|
| **預設安全** | 沒有任何 flag 時，所有限制取交集（最嚴格），確保不超配 |
| **顯式授權** | 覆蓋需要管理員主動設定 flag，不會意外發生 |
| **會計不可繞過** | 即使覆蓋了 Partition，QOS 和 Association 的會計限制仍然生效 |
| **分層防禦** | Partition 是第一道門，QOS 會計是第二道門，任何一道都能擋住 |

Partition 限制代表「物理分區的建議邊界」，QOS 會計限制代表「組織的資源策略」。前者可以在需要時彈性調整，後者則是不可動搖的底線。

---

## 相關原始碼索引

| 函式 | 檔案位置 | 職責 |
|---|---|---|
| `_qos_part_check()` | `src/slurmctld/job_mgr.c:6395` | 檢查作業請求是否超過 Partition 節點限制 |
| `_part_access_check()` | `src/slurmctld/job_mgr.c:6452` | 整合 Partition 存取權限與限制檢查 |
| `_valid_job_part()` | `src/slurmctld/job_mgr.c:6748` | 根據 `EnforcePartLimits` 決定是否拒絕提交 |
| `_foreach_valid_part()` | `src/slurmctld/job_mgr.c:6699` | 多 partition 逐一檢查（`ANY` vs `ALL` 差異實現） |
| `job_limits_check()` | `src/slurmctld/job_mgr.c:6953` | 排程器持續檢查，設定 pending reason |
| `_job_runnable_test2()` | `src/slurmctld/job_scheduler.c:404` | 排程器呼叫 `job_limits_check()` 的入口 |
| `get_node_cnts()` | `src/slurmctld/node_scheduler.c:3144` | 排程決策階段節點數計算 |
| `acct_policy_validate()` | `src/slurmctld/acct_policy.c:3635` | 提交時 QOS/Association TRES 限制驗證入口 |
| `_qos_policy_validate()` | `src/slurmctld/acct_policy.c:1685` | 提交時 QOS TRES 限制檢查（受 `DenyOnLimit` 控制） |
| `_validate_tres_limits_for_qos()` | `src/slurmctld/acct_policy.c:1336` | TRES 靜態限制驗證（`strict_checking` 閘門） |
| `_qos_job_runnable_post_select()` | `src/slurmctld/acct_policy.c:2326` | 排程時 QOS TRES 使用量檢查（含 `MaxTRESPerUser`） |
| `acct_policy_get_max_nodes()` | `src/slurmctld/acct_policy.c:4402` | 會計政策最大節點數查詢 |
| `acct_policy_set_qos_order()` | `src/slurmctld/acct_policy.c:5254` | 雙 QOS 優先順序決策 |
| `parse_part_enforce_type()` | `src/common/slurm_protocol_defs.c:5661` | `EnforcePartLimits` 設定值解析 |
| `_validate_accounting_storage_enforce()` | `src/common/read_config.c:3420` | `AccountingStorageEnforce` 設定值解析 |
| `ACCOUNTING_ENFORCE_*` 定義 | `src/common/read_config.h:66-73` | 會計強制旗標位元定義 |
| `QOS_FLAG_*` 定義 | `slurm/slurmdb.h:126-133` | QOS 旗標位元定義 |

---

## QOS Flag 快速參考

| Flag 名稱 | sacctmgr 設定值 | 效果 |
|---|---|---|
| `QOS_FLAG_PART_MIN_NODE` | `PartitionMinNodes` | 允許作業的 min_nodes 低於 Partition MinNodes |
| `QOS_FLAG_PART_MAX_NODE` | `PartitionMaxNodes` | 允許作業的 max_nodes 超過 Partition MaxNodes |
| `QOS_FLAG_PART_TIME_LIMIT` | `PartitionTimeLimit` | 允許作業的時間限制超過 Partition MaxTime |
| `QOS_FLAG_OVER_PART_QOS` | `OverPartQOS` | Job QOS 優先於 Partition QOS（改變會計限制的優先順序） |
| `QOS_FLAG_DENY_LIMIT` | `DenyOnLimit` | 提交時嚴格檢查 TRES 限制，超過則拒絕。未設定時只在排程階段 PENDING hold |
