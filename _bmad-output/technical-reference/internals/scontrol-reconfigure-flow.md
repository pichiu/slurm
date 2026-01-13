---
title: scontrol reconfigure 內部實現解析
description: 深入剖析 Slurm 配置重載的完整程式碼流程、Fork 機制與子系統重建過程
author: Technical Writer (Paige)
date: 2026-01-12
category: internals
---

# scontrol reconfigure 內部實現解析

## 概述

本文件深入剖析 Slurm 中 `scontrol reconfigure` 命令的完整內部實現流程。這是一個 **重量級操作**，會透過 Fork + execve 機制啟動新的 slurmctld 進程來取代舊進程，確保配置變更的完整性和一致性。

> **讀者對象**：Slurm 核心開發者、系統管理員、DevOps 工程師
>
> **先備知識**：建議先了解 [Slurm Partition 配置指南](../configuration/slurm-partition-guide.md) 與 Unix 進程管理概念。

### 本文涵蓋內容

- 完整的 RPC 到進程切換流程
- Fork + execve 機制詳解
- `read_slurm_conf()` 配置重載核心邏輯
- 不可動態變更的配置項目
- 與動態操作（如 `scontrol create partition`）的比較

---

## 目錄

- [一、整體架構](#一整體架構)
- [二、核心技術概念](#二核心技術概念)
- [三、客戶端流程](#三客戶端流程)
- [四、slurmctld 處理流程](#四slurmctld-處理流程)
- [五、Fork + Execve 機制](#五fork--execve-機制)
- [六、配置重載核心 - read_slurm_conf()](#六配置重載核心---read_slurm_conf)
- [七、不可變更的配置項目](#七不可變更的配置項目)
- [八、通知 slurmd 節點](#八通知-slurmd-節點)
- [九、與動態操作的比較](#九與動態操作的比較)
- [十、完整函數調用鏈](#十完整函數調用鏈)
- [十一、錯誤處理與故障排除](#十一錯誤處理與故障排除)
- [延伸閱讀](#延伸閱讀)

---

## 一、整體架構

### 1.1 系統層級視圖

`scontrol reconfigure` 的處理流程跨越多個系統層級：

```mermaid
flowchart TB
    subgraph Client["客戶端層"]
        A1[scontrol reconfigure] --> A2[slurm_reconfigure API]
    end

    subgraph Network["網路層"]
        B1[REQUEST_RECONFIGURE RPC]
    end

    subgraph OldCtld["舊 slurmctld 進程"]
        C1[RPC Handler] --> C2[權限驗證]
        C2 --> C3[發送 SIGHUP]
        C3 --> C4[_attempt_reconfig]
        C4 --> C5[Fork 子進程]
    end

    subgraph NewCtld["新 slurmctld 進程"]
        D1[execve 啟動] --> D2[read_slurm_conf]
        D2 --> D3[重建所有資料結構]
        D3 --> D4[通知啟動成功]
    end

    subgraph Nodes["計算節點層"]
        E1[slurmd 1]
        E2[slurmd 2]
        E3[slurmd N]
    end

    A2 --> B1
    B1 --> C1
    C5 --> D1
    D4 --> |舊進程退出| C5
    D3 --> |REQUEST_RECONFIGURE| E1
    D3 --> |REQUEST_RECONFIGURE| E2
    D3 --> |REQUEST_RECONFIGURE| E3
```

### 1.2 完整序列圖

```mermaid
sequenceDiagram
    autonumber
    participant Admin as 管理員
    participant SC as scontrol
    participant API as libslurm API
    participant Old as slurmctld (舊)
    participant New as slurmctld (新)
    participant Slurmd as slurmd 節點群

    Admin->>SC: scontrol reconfigure

    rect rgb(230, 245, 255)
        Note over SC,API: 客戶端處理
        SC->>API: slurm_reconfigure()
        API->>Old: REQUEST_RECONFIGURE RPC
    end

    rect rgb(255, 245, 230)
        Note over Old: 舊進程處理階段
        Old->>Old: validate_super_user()
        Old->>Old: reconfigure_slurm(msg)
        Old->>Old: 加入 reconfig_reqs 佇列
        Old->>Old: pthread_kill(SIGHUP)
        Old->>Old: _on_sighup() 設定 reconfig=true
        Old->>Old: slurmctld_shutdown()
        Old->>Old: conmgr_quiesce() 暫停連線
    end

    rect rgb(230, 255, 230)
        Note over Old,New: 進程切換階段
        Old->>Old: _try_to_reconfig()
        Old->>New: fork()
        New->>New: execve(slurmctld binary)
        New->>New: 檢測 SLURMCTLD_RECONF 環境變數
        New->>New: read_slurm_conf(recover=2)
    end

    rect rgb(255, 230, 255)
        Note over New: 新進程初始化
        New->>New: build_all_nodeline_info()
        New->>New: _build_all_partitionline_info()
        New->>New: load_all_node_state()
        New->>New: load_all_job_state()
        New->>New: load_all_part_state()
        New->>New: load_all_resv_state()
        New->>New: controller_reconfig_scheduling()
        New->>New: gs_reconfig()
    end

    New-->>Old: 寫入 PID 到 pipe（通知成功）
    Old->>Old: _exit(0) 結束舊進程

    rect rgb(230, 255, 245)
        Note over New,Slurmd: 通知節點階段
        New->>Slurmd: REQUEST_RECONFIGURE / configless_update
        Slurmd->>Slurmd: 重新讀取 slurm.conf
    end

    New-->>API: SLURM_SUCCESS
    API-->>SC: 返回結果
    SC-->>Admin: 完成
```

---

## 二、核心技術概念

### 2.1 為什麼使用 Fork + Execve？

Slurm 選擇使用 **Fork + Execve** 機制而非原地重載配置，主要原因：

```mermaid
flowchart TD
    subgraph Problem["原地重載的問題"]
        P1[記憶體洩漏風險]
        P2[插件狀態不一致]
        P3[全域變數殘留]
        P4[無法回滾]
    end

    subgraph Solution["Fork + Execve 的優勢"]
        S1[乾淨的記憶體空間]
        S2[插件完整重新初始化]
        S3[全域變數重設]
        S4[失敗時舊進程可恢復]
    end

    Problem -->|解決| Solution
```

**程式碼說明**（`controller.c:1456-1510`）：

```c
static int _try_to_reconfig(void)
{
    pid_t pid;

    /* Fork 子進程 */
    if ((pid = fork()) < 0) {
        fatal("%s: fork() failed: %m", __func__);
    } else if (pid > 0) {
        /* 父進程（舊 slurmctld）*/
        /* 等待子進程通知啟動成功 */
        safe_read(to_parent[0], &grandchild_pid, sizeof(pid_t));
        info("Relinquishing control to new slurmctld process");
        return SLURM_SUCCESS;  /* 之後會 _exit(0) */
    }

    /* 子進程執行新的 slurmctld */
    execve(binary, main_argv, child_env);
    fatal("execv() failed: %m");
}
```

### 2.2 SIGHUP 信號機制

在 Unix 系統中，SIGHUP 傳統上用於通知進程「掛起」（Hang Up）。Slurm 將其重新定義為「重新配置」信號：

```mermaid
stateDiagram-v2
    [*] --> Running: slurmctld 啟動
    Running --> SIGHUP_Received: 收到 SIGHUP
    SIGHUP_Received --> Shutdown: 設定 reconfig=true
    Shutdown --> Quiesce: 暫停接收連線
    Quiesce --> Fork: _try_to_reconfig()
    Fork --> NewProcess: execve()
    Fork --> OldExit: 新進程成功
    OldExit --> [*]
    NewProcess --> Running: 初始化完成
```

**程式碼說明**（`controller.c:396-416`）：

```c
static void _on_sighup(conmgr_callback_args_t conmgr_args, void *arg)
{
    info("Reconfigure signal (SIGHUP) received");

    /* 檢查是否在備援模式 */
    if (standby_mode) {
        backup_on_sighup();
        return;
    }

    /* 設定重新配置標誌 */
    reconfig = true;

    /* 觸發控制器關閉流程 */
    slurmctld_shutdown();
}
```

### 2.3 進程間通訊（IPC）

父子進程透過 **pipe** 進行通訊，確保新進程成功啟動後舊進程才退出：

```mermaid
sequenceDiagram
    participant Parent as 父進程 (舊)
    participant Pipe as Pipe 管道
    participant Child as 子進程 (新)

    Parent->>Pipe: pipe() 建立管道
    Parent->>Child: fork()

    Note over Parent: 關閉寫入端 to_parent[1]
    Note over Child: 保留寫入端 to_parent[1]

    Child->>Child: execve() 啟動新 slurmctld
    Child->>Child: read_slurm_conf() 初始化

    alt 初始化成功
        Child->>Pipe: write(pid) 寫入 PID
        Pipe->>Parent: read(pid) 讀取 PID
        Parent->>Parent: _exit(0) 正常退出
    else 初始化失敗
        Child->>Child: fatal() 異常退出
        Pipe->>Parent: read() 返回 0（EOF）
        Parent->>Parent: 恢復運行，繼續服務
    end
```

---

## 三、客戶端流程

### 3.1 命令入口

當管理員執行 `scontrol reconfigure` 時：

**檔案**：`src/scontrol/scontrol.c`

```c
/* 識別 reconfigure 命令 */
if (xstrncasecmp(cmd, "reconfigure", MAX(cmdlen, 6)) == 0) {
    if (slurm_reconfigure()) {
        exit_code = 1;
        slurm_perror("slurm_reconfigure error");
    }
}
```

### 3.2 API 層

**檔案**：`src/api/reconfigure.c:65-82`

```c
int slurm_reconfigure(void)
{
    int rc = SLURM_SUCCESS;
    slurm_msg_t req;

    slurm_msg_t_init(&req);
    req.msg_type = REQUEST_RECONFIGURE;

    /* 發送 RPC 到 slurmctld */
    if (slurm_send_recv_controller_rc_msg(&req, &rc,
                                          working_cluster_rec) < 0)
        return SLURM_ERROR;

    if (rc)
        slurm_seterrno(rc);

    return rc ? SLURM_ERROR : SLURM_SUCCESS;
}
```

---

## 四、slurmctld 處理流程

### 4.1 RPC 分派表

**檔案**：`src/slurmctld/proc_req.c:6711`

```c
{
    .msg_type = REQUEST_RECONFIGURE,
    .func = _slurm_rpc_reconfigure_controller,
}
```

### 4.2 RPC 處理函數

**檔案**：`src/slurmctld/proc_req.c:3184-3198`

```c
static void _slurm_rpc_reconfigure_controller(slurm_msg_t *msg)
{
    /* 步驟 1：權限驗證 - 必須是超級用戶 */
    if (!validate_super_user(msg->auth_uid)) {
        error("Security violation, RECONFIGURE RPC from uid=%u",
              msg->auth_uid);
        slurm_send_rc_msg(msg, ESLURM_USER_ID_MISSING);

        /* 清理連線資源 */
        conn_g_destroy(msg->conn, true);
        msg->conn = NULL;
        FREE_NULL_MSG(msg);
        return;
    }

    info("Processing Reconfiguration Request");

    /* 步驟 2：開始重新配置流程 */
    reconfigure_slurm(msg);
}
```

### 4.3 觸發重新配置

**檔案**：`src/slurmctld/controller.c:1552-1559`

```c
extern void reconfigure_slurm(slurm_msg_t *msg)
{
    xassert(msg);

    /* 將請求加入佇列，稍後回覆 */
    list_append(reconfig_reqs, msg);

    /* 發送 SIGHUP 給自己，觸發重新配置流程 */
    pthread_kill(pthread_self(), SIGHUP);
}
```

---

## 五、Fork + Execve 機制

### 5.1 嘗試重新配置

**檔案**：`src/slurmctld/controller.c:324-359`

```c
static void _attempt_reconfig(void)
{
    info("Attempting to reconfigure");

    /* 步驟 1：暫停所有連線處理 */
    conmgr_quiesce(__func__);

    /* 步驟 2：在前台模式下先回覆請求者 */
    if (!daemonize && !under_systemd)
        _send_reconfig_replies();

    /* 步驟 3：嘗試 Fork 新進程 */
    reconfig_rc = _try_to_reconfig();

    /* 步驟 4：回覆所有等待的請求 */
    _send_reconfig_replies();

    /* 步驟 5：根據結果決定後續動作 */
    if (!reconfig_rc) {
        info("Relinquishing control to new child");
        _exit(0);  /* 新進程成功，舊進程退出 */
    }

    /* 重新配置失敗，恢復連線處理 */
    recover = 2;
    conmgr_unquiesce(__func__);
}
```

### 5.2 Fork 新進程的詳細流程

**檔案**：`src/slurmctld/controller.c:1390-1512`

```mermaid
flowchart TD
    A[_try_to_reconfig 開始] --> B[準備子進程環境變數]
    B --> C["設定 SLURMCTLD_RECONF=1"]
    C --> D{運行模式?}

    D -->|前台模式| E[slurmscriptd_fini]
    D -->|背景模式| F[建立 pipe]

    E --> G[直接執行 start_child]
    F --> H[fork]

    H --> I{fork 結果}
    I -->|父進程 pid > 0| J[關閉 pipe 寫入端]
    I -->|子進程 pid = 0| K[進入 start_child]

    J --> L[等待讀取子進程 PID]
    L --> M{讀取成功?}
    M -->|是| N[返回 SUCCESS]
    M -->|否| O[返回 ERROR，繼續運行]

    K --> P[關閉其他 fd]
    P --> Q{systemd 模式?}
    Q -->|是| R[再次 fork 確保 init 為父進程]
    Q -->|否| S[execve 執行新 slurmctld]
    R --> S

    N --> T[舊進程 _exit 0]
    O --> U[舊進程恢復運行]
```

### 5.3 環境變數傳遞

新進程透過環境變數識別自己是重新配置產生的：

| 環境變數 | 說明 |
|----------|------|
| `SLURMCTLD_RECONF=1` | 標識這是重新配置啟動 |
| `SLURMCTLD_RECONF_PIDFD` | 傳遞 PID 檔案描述符 |
| `SLURMCTLD_RECONF_PARENT_FD` | 父進程通訊管道 |
| `SLURMCTLD_RECONF_LISTEN_FDS` | 監聽 socket 列表 |
| `SLURMCTLD_RECONF_LISTEN_COUNT` | 監聽 socket 數量 |

---

## 六、配置重載核心 - read_slurm_conf()

### 6.1 函數概述

**檔案**：`src/slurmctld/read_config.c:1556-1900+`

`read_slurm_conf()` 是配置重載的核心函數，負責重建 slurmctld 的所有資料結構。

```c
/*
 * read_slurm_conf - load the slurm configuration from the configured file.
 * IN recover - 恢復模式：
 *              0 = 從 slurm.conf 完全重建
 *              1 = 恢復作業和觸發器狀態，節點 DOWN/DRAIN 狀態
 *              2 = 恢復所有狀態（reconfigure 使用此模式）
 */
extern int read_slurm_conf(int recover)
```

### 6.2 執行流程

```mermaid
flowchart TD
    A[read_slurm_conf 開始] --> B[保存舊插件類型]

    subgraph Init["初始化階段"]
        B --> C[_init_all_slurm_conf]
        C --> D[cgroup_conf_init]
        D --> E[topology_g_init]
    end

    subgraph BuildNodes["節點建構階段"]
        E --> F[build_all_nodeline_info]
        F --> G[grow_node_record_table_ptr]
        G --> H[_handle_all_downnodes]
    end

    subgraph BuildParts["Partition 建構階段"]
        H --> I[_build_all_partitionline_info]
        I --> J[load_config_state_lite]
    end

    subgraph LoadState["狀態載入階段"]
        J --> K{recover 模式?}
        K -->|0| L[從 slurm.conf 完全重建]
        K -->|1| M[load_all_node_state 部分]
        K -->|2| N[load_all_node_state 完整]

        L --> O[load_last_job_id]
        M --> P[load_all_job_state]
        N --> P
        O --> P
        P --> Q[load_all_part_state]
        Q --> R[load_all_resv_state]
    end

    subgraph Rebuild["重建階段"]
        R --> S[_build_bitmaps]
        S --> T[_build_part_bitmaps]
        T --> U[select_g_node_init]
        U --> V[config_power_mgr]
    end

    subgraph Sync["同步階段"]
        V --> W[_sync_jobs_to_conf]
        W --> X[_sync_nodes_to_jobs]
        X --> Y[license_update]
        Y --> Z[set_cluster_tres]
    end

    subgraph Validate["驗證階段"]
        Z --> AA[驗證插件變更]
        AA --> AB{插件可變更?}
        AB -->|否| AC[還原舊設定，返回錯誤]
        AB -->|是| AD[controller_reconfig_scheduling]
        AD --> AE[gs_reconfig]
        AE --> AF[返回成功]
    end
```

### 6.3 關鍵子步驟解析

#### 6.3.1 節點資訊重建

```c
/* 從 slurm.conf 建構節點資訊 */
if ((error_code = build_all_nodeline_info(false, slurmctld_tres_cnt)))
    goto end_it;

/* 擴展節點表以支援動態節點 */
if ((slurm_conf.max_node_cnt != NO_VAL) &&
    node_record_count < slurm_conf.max_node_cnt) {
    node_record_count = slurm_conf.max_node_cnt;
    grow_node_record_table_ptr();
}
```

#### 6.3.2 狀態載入（recover=2 模式）

```c
if (recover > 1) {
    /* 完整恢復所有狀態 */
    (void) load_all_node_state(false);   /* 節點狀態 */
    _set_features(NULL, 0, recover);      /* 特性設定 */
}

/* 載入作業狀態 */
reconfig_flags |= RECONFIG_KEEP_PART_INFO;
load_all_job_state();

/* 載入 Partition 和預約狀態 */
(void) load_all_part_state(reconfig_flags);
load_all_resv_state(recover);
```

#### 6.3.3 排程器重新配置

```c
if (recover >= 1) {
    trigger_state_restore();
    controller_reconfig_scheduling();  /* 重新配置排程器 */
}

/* Gang Scheduler 重新配置 */
gs_reconfig();
```

---

## 七、不可變更的配置項目

某些核心配置在 `scontrol reconfigure` 時 **不能變更**，必須完全重啟 slurmctld：

### 7.1 不可變更配置列表

| 配置項目 | 錯誤碼 | 原因 |
|----------|--------|------|
| `AuthType` | `ESLURM_INVALID_AUTHTYPE_CHANGE` | 認證機制改變會導致現有連線失效 |
| `BurstBufferType` | `ESLURM_INVALID_BURST_BUFFER_CHANGE` | Burst Buffer 狀態無法遷移 |
| `CredType` | `ESLURM_INVALID_CRED_TYPE_CHANGE` | 憑證類型改變會導致作業無法驗證 |
| `NamespacePlugin` | `ESLURM_INVALID_NAMESPACE_CHANGE` | 命名空間隔離機制無法動態切換 |
| `SchedType` | `ESLURM_INVALID_SCHEDTYPE_CHANGE` | 排程器狀態無法遷移 |
| `SelectType` | `ESLURM_INVALID_SELECTTYPE_CHANGE` | 資源選擇邏輯改變會影響現有分配 |
| `SwitchType` | `ESLURM_INVALID_SWITCHTYPE_CHANGE` | 網路交換器插件無法動態切換 |

### 7.2 驗證邏輯

**檔案**：`src/slurmctld/read_config.c:1796-1843`

```c
/* 驗證 AuthType 是否變更 */
if (xstrcmp(old_auth_type, slurm_conf.authtype)) {
    xfree(slurm_conf.authtype);
    slurm_conf.authtype = old_auth_type;  /* 還原舊設定 */
    old_auth_type = NULL;
    rc = ESLURM_INVALID_AUTHTYPE_CHANGE;
}

/* 驗證 SelectType 是否變更 */
if (xstrcmp(old_select_type, slurm_conf.select_type)) {
    xfree(slurm_conf.select_type);
    slurm_conf.select_type = old_select_type;
    old_select_type = NULL;
    rc = ESLURM_INVALID_SELECTTYPE_CHANGE;
}

/* ... 其他驗證 ... */
```

### 7.3 處理流程

```mermaid
flowchart TD
    A[read_slurm_conf] --> B[保存舊插件類型]
    B --> C[讀取新 slurm.conf]
    C --> D[比較插件類型]

    D --> E{AuthType 變更?}
    E -->|是| F[還原舊值，設定錯誤碼]
    E -->|否| G{SelectType 變更?}

    F --> G
    G -->|是| H[還原舊值，設定錯誤碼]
    G -->|否| I{SchedType 變更?}

    H --> I
    I -->|是| J[還原舊值，設定錯誤碼]
    I -->|否| K[繼續其他驗證...]

    J --> K
    K --> L{有任何錯誤?}
    L -->|是| M[記錄警告，返回非致命錯誤]
    L -->|否| N[配置載入成功]
```

---

## 八、通知 slurmd 節點

### 8.1 通知機制

配置重載成功後，slurmctld 會通知所有 slurmd 節點：

**檔案**：`src/slurmctld/controller.c:1561-1570`

```c
static void _post_reconfig(void)
{
    if (running_configless) {
        /* Configless 模式：主動推送配置到節點 */
        configless_update();
        push_reconfig_to_slurmd();
        sackd_mgr_push_reconfig();
    } else {
        /* 傳統模式：通知節點重新讀取本地 slurm.conf */
        msg_to_slurmd(REQUEST_RECONFIGURE);
    }
}
```

### 8.2 兩種模式比較

```mermaid
flowchart LR
    subgraph Traditional["傳統模式"]
        T1[slurmctld] -->|REQUEST_RECONFIGURE| T2[slurmd]
        T2 --> T3[讀取本地 slurm.conf]
    end

    subgraph Configless["Configless 模式"]
        C1[slurmctld] -->|推送配置內容| C2[slurmd]
        C2 --> C3[使用推送的配置]
    end
```

| 特性 | 傳統模式 | Configless 模式 |
|------|----------|----------------|
| 配置來源 | 各節點本地 slurm.conf | slurmctld 推送 |
| 一致性 | 依賴檔案同步工具 | 自動保證一致 |
| 網路開銷 | 低（僅發送通知） | 較高（傳輸配置內容） |
| 適用場景 | 傳統叢集 | 雲端/容器化環境 |

### 8.3 slurmd 處理

**檔案**：`src/slurmd/slurmd/req.c:5032-5043`

```c
{
    .msg_type = REQUEST_RECONFIGURE,
    .func = _rpc_reconfigure,
},
{
    .msg_type = REQUEST_RECONFIGURE_WITH_CONFIG,
    .func = _rpc_reconfigure,
}
```

---

## 九、與動態操作的比較

### 9.1 比較表

| 特性 | `scontrol reconfigure` | `scontrol create partition/node` |
|------|------------------------|----------------------------------|
| **機制** | Fork + execve 新進程 | 直接修改記憶體 |
| **是否重啟** | 是（新進程取代舊進程） | 否 |
| **讀取 slurm.conf** | 是（完整重讀） | 否 |
| **影響範圍** | 整個叢集 | 僅該資源 |
| **中斷服務** | 短暫（fork 期間） | 無 |
| **通知 slurmd** | 是 | 否 |
| **可變更插件** | 否（部分插件） | 不適用 |
| **適用場景** | 修改 slurm.conf 後 | 動態增減資源 |

### 9.2 選擇指南

```mermaid
flowchart TD
    A[需要變更配置] --> B{變更類型?}

    B -->|新增 Partition| C[scontrol create partition]
    B -->|新增節點| D[scontrol create node]
    B -->|修改現有設定| E{設定類型?}

    E -->|Partition 屬性| F[scontrol update partition]
    E -->|節點屬性| G[scontrol update node]
    E -->|全域設定| H[修改 slurm.conf]

    H --> I[scontrol reconfigure]

    C --> J[立即生效，無需重啟]
    D --> J
    F --> J
    G --> J
    I --> K[Fork 新進程，短暫中斷]
```

### 9.3 `trigger_reconfig()` 的澄清

**重要**：`scontrol create node` 中呼叫的 `trigger_reconfig()` **不是** `scontrol reconfigure`！

```c
/* src/slurmctld/trigger_mgr.c:575-584 */
extern void trigger_reconfig(void)
{
    /* 這只是設定一個標誌，用於 Slurm 觸發器系統 */
    slurm_mutex_lock(&trigger_mutex);
    trigger_node_reconfig = true;  /* 通知已註冊的觸發器腳本 */
    slurm_mutex_unlock(&trigger_mutex);
}
```

這個函數只是通知 Slurm 的 [Trigger 系統](https://slurm.schedmd.com/strigger.html)，讓用戶註冊的觸發器腳本可以執行，與 `scontrol reconfigure` 完全無關。

---

## 十、完整函數調用鏈

```
管理員執行: scontrol reconfigure
│
├─► scontrol [src/scontrol/scontrol.c]
│   └─► slurm_reconfigure() [src/api/reconfigure.c:65]
│       └─► slurm_send_recv_controller_rc_msg()
│           │
│ ========================= 網路傳輸 RPC =========================
│           │
│           ▼ REQUEST_RECONFIGURE
│
├─► slurmctld RPC 分派表 [proc_req.c:6711]
│   └─► _slurm_rpc_reconfigure_controller() [proc_req.c:3184]
│       ├─► validate_super_user() - 權限驗證
│       └─► reconfigure_slurm() [controller.c:1552]
│           ├─► list_append(reconfig_reqs, msg)
│           └─► pthread_kill(pthread_self(), SIGHUP)
│
├─► _on_sighup() [controller.c:396]
│   ├─► reconfig = true
│   └─► slurmctld_shutdown()
│
├─► 主迴圈檢測 reconfig [controller.c:1132]
│   └─► _attempt_reconfig() [controller.c:324]
│       ├─► conmgr_quiesce() - 暫停連線
│       ├─► _send_reconfig_replies() - 回覆請求者（前台模式）
│       └─► _try_to_reconfig() [controller.c:1390]
│           │
│           ├─► 準備環境變數
│           │   ├─► SLURMCTLD_RECONF=1
│           │   ├─► SLURMCTLD_RECONF_PIDFD
│           │   └─► SLURMCTLD_RECONF_PARENT_FD
│           │
│           ├─► pipe(to_parent) - 建立 IPC 管道
│           │
│           └─► fork()
│               │
│               ├─► [父進程] pid > 0
│               │   ├─► close(to_parent[1])
│               │   ├─► safe_read(to_parent[0], &grandchild_pid)
│               │   ├─► waitpid(pid) - 等待子進程
│               │   └─► 返回 SUCCESS
│               │
│               └─► [子進程] pid = 0
│                   ├─► closeall_except() - 關閉其他 fd
│                   ├─► [systemd] 再次 fork() 確保父進程是 init
│                   └─► execve(binary, main_argv, child_env)
│
│ ========================= 新進程啟動 =========================
│
├─► main() [controller.c] - 新 slurmctld
│   ├─► 檢測 SLURMCTLD_RECONF 環境變數
│   ├─► reconfiguring = true
│   │
│   └─► read_slurm_conf(recover=2) [read_config.c:1556]
│       │
│       ├─► 保存舊插件類型
│       │
│       ├─► _init_all_slurm_conf()
│       ├─► cgroup_conf_init()
│       ├─► topology_g_init()
│       │
│       ├─► build_all_nodeline_info() - 建構節點
│       ├─► _build_all_partitionline_info() - 建構 Partition
│       │
│       ├─► load_all_node_state() - 載入節點狀態
│       ├─► load_all_job_state() - 載入作業狀態
│       ├─► load_all_part_state() - 載入 Partition 狀態
│       ├─► load_all_resv_state() - 載入預約狀態
│       │
│       ├─► _build_bitmaps() - 建立位圖
│       ├─► _build_part_bitmaps() - 建立 Partition 位圖
│       ├─► select_g_node_init() - 選擇插件初始化
│       ├─► config_power_mgr() - 電源管理配置
│       │
│       ├─► _sync_jobs_to_conf() - 同步作業
│       ├─► _sync_nodes_to_jobs() - 同步節點
│       │
│       ├─► 驗證插件變更（不可變更的會還原）
│       │
│       ├─► controller_reconfig_scheduling() - 排程器重配置
│       └─► gs_reconfig() - Gang Scheduler 重配置
│
├─► notify_parent_of_success() [controller.c:1514]
│   └─► safe_write(fd, &pid) - 通知父進程
│
├─► [舊進程] 收到通知後 _exit(0)
│
└─► _post_reconfig() [controller.c:1561]
    ├─► [Configless] configless_update() + push_reconfig_to_slurmd()
    └─► [傳統] msg_to_slurmd(REQUEST_RECONFIGURE)
        └─► 所有 slurmd 重新讀取配置
```

---

## 十一、錯誤處理與故障排除

### 11.1 常見錯誤

| 錯誤 | 原因 | 解決方案 |
|------|------|----------|
| `ESLURM_USER_ID_MISSING` | 非 root 或 SlurmUser 執行 | 使用正確的管理員帳號 |
| `ESLURM_INVALID_AUTHTYPE_CHANGE` | 嘗試變更 AuthType | 完全重啟 slurmctld |
| `ESLURM_INVALID_SELECTTYPE_CHANGE` | 嘗試變更 SelectType | 完全重啟 slurmctld |
| Fork 失敗 | 系統資源不足 | 檢查 ulimit 和記憶體 |
| Execve 失敗 | slurmctld binary 問題 | 驗證 binary 路徑和權限 |

### 11.2 日誌訊息解讀

```bash
# 正常流程
info("Processing Reconfiguration Request")
info("Reconfigure signal (SIGHUP) received")
info("Attempting to reconfigure")
info("Relinquishing control to new slurmctld process")
info("child started successfully")

# 失敗情況
info("Resuming operation, reconfigure failed.")
error("failed to notify parent, may have two processes running now")
```

### 11.3 除錯技巧

```bash
# 1. 檢查 slurmctld 日誌
tail -f /var/log/slurm/slurmctld.log | grep -i reconfig

# 2. 確認進程狀態
ps aux | grep slurmctld

# 3. 驗證配置語法（不重載）
slurmctld -t

# 4. 查看當前配置
scontrol show config

# 5. 比較新舊配置
diff /etc/slurm/slurm.conf /etc/slurm/slurm.conf.bak
```

---

## 延伸閱讀

### 相關技術文件

| 文件 | 說明 |
|------|------|
| [Slurm Partition 完整技術指南](../configuration/slurm-partition-guide.md) | Partition 配置參數與實務範例 |
| [scontrol create partition 內部實現](./scontrol-create-partition-flow.md) | 動態建立 Partition 的程式碼流程 |

### 官方參考資源

| 資源 | 說明 |
|------|------|
| [SchedMD scontrol 文件](https://slurm.schedmd.com/scontrol.html) | scontrol 命令官方文件 |
| [Slurm 配置指南](https://slurm.schedmd.com/slurm.conf.html) | slurm.conf 完整參數說明 |
| [Slurm Trigger 文件](https://slurm.schedmd.com/strigger.html) | 觸發器系統說明 |

---

*本文件基於 Slurm 原始碼分析撰寫，如有版本差異請以實際原始碼為準。*
