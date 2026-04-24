# Slurm 指標指南

## TL;DR

Slurm 25.11 起透過 slurmctld 的 HTTP 端點（預設連接埠 6817）對外提供叢集指標（Metrics），支援 Prometheus 等監控系統。需在 `slurm.conf` 設定 `MetricsType=metrics/openmetrics`，且不得啟用任何 `PrivateData` 參數。指標涵蓋作業、節點、分割區與排程器等類別，使用時須注意安全性與效能影響。

---

## 翻譯

### 目錄

- [指標概觀](#指標概觀)
- [設定](#設定)
- [HTTP 端點](#http-端點)
- [OpenMetrics 外掛程式](#openmetrics-外掛程式)
- [Slurm 提供的指標類別](#slurm-提供的指標類別)
- [安全性考量](#安全性考量)
- [效能影響](#效能影響)
- [使用範例](#使用範例)

---

### 指標概觀

Slurm 25.11 引入了一套完整的系統，用於收集並對外公開與叢集資源、作業狀態及排程器效能相關的指標（Metrics）。指標系統透過 slurmctld 守護程式提供的 HTTP 端點，即時揭露各種 Slurm 實體的資料。

此指標功能可與 Prometheus、Grafana 及其他可觀測性工具等主流監控系統整合。

---

### 設定

#### 前置條件

指標功能需要在 `slurm.conf` 中進行特定設定：

- **`PrivateData` 不得設定**：若 `slurm.conf` 中設定了任何 `PrivateData` 參數，指標功能將自動停用。這是為了防止透過指標洩漏敏感資訊的安全性要求。

- **`MetricsType` 參數**：設定 [`MetricsType`](slurm.conf.html#OPT_MetricsType) 參數以指定要使用的指標外掛程式。目前僅支援 OpenMetrics 外掛程式：

```
MetricsType=metrics/openmetrics
```

#### 外掛程式載入

當 `MetricsType` 參數已設定時，slurmctld 會自動載入指標外掛程式。

---

### HTTP 端點

Slurm 透過 slurmctld 守護程式的監聽連接埠（預設 6817）上的 HTTP GET 端點對外提供指標。可用的端點如下：

- **GET /metrics** — 列出可用的指標端點
- **GET /metrics/jobs** — 作業相關指標，包含依狀態的數量統計、資源分配及作業統計（[範例](#作業指標範例)）
- **GET /metrics/jobs-users-accts** — 依使用者與帳號細分的作業指標（[範例](#使用者與帳號作業指標範例)）
- **GET /metrics/nodes** — 節點相關指標，包含資源數量、節點狀態及使用率（[範例](#節點指標範例)）
- **GET /metrics/partitions** — 分割區（Partition）相關指標，包含每個分割區的作業數量及資源分配（[範例](#分割區指標範例)）
- **GET /metrics/scheduler** — 排程器效能指標，包含循環時間、回填（Backfill）統計及佇列長度（[範例](#排程器指標範例)）

所有端點均以 UTF-8 文字格式回傳資料，與 Prometheus 及其他監控系統相容。

---

### OpenMetrics 外掛程式

OpenMetrics 外掛程式實作了 [OpenMetrics 1.0](https://openmetrics.io/) 規範，確保與 Prometheus 及其他使用此格式的監控系統相容。

#### 指標格式

每筆指標遵循 OpenMetrics 格式，包含以下元件：

- **指標名稱（Metric name）**：以 `slurm_` 為前綴的描述性名稱
- **指標類型（Metric type）**：僅公開 `gauge`（量測值）類型的指標
- **指標值（Metric value）**：實際數值
- **標籤（Labels）**：用於附加情境的選用鍵值對
- **說明文字（Help text）**：指標的人類可讀描述

---

### Slurm 提供的指標類別

每個端點提供一組與相同大類相關的指標。由於指標數量眾多，本頁並未全部列出，以下各小節僅提供各類別的部分範例。

#### 作業指標

作業指標提供作業狀態、資源分配及作業數量等資訊。範例包含：

- `slurm_jobs` — 作業總數
- `slurm_jobs_running` — 執行中的作業數量
- `slurm_jobs_pending` — 等待中的作業數量（請見下方注意事項）
- `slurm_jobs_cpus_alloc` — 已分配給作業的 CPU 總數
- `slurm_jobs_memory_alloc` — 已分配給作業的記憶體總量

**注意**：在 Slurm 中，等待中的作業（Pending jobs）包含等待資源的作業以及被暫停（Hold）的作業。被暫停的作業在解除暫停前不會被排程。

#### 使用者與帳號作業指標

依使用者與帳號細分的作業指標，提供系統中每位活躍使用者及帳號在各狀態下的作業數量，以鍵值對方式儲存每個實體。請注意，每個唯一的鍵值對都代表一個新的時間序列（Time Series），可能大幅增加儲存的資料量。範例包含：

- `slurm_user_jobs_pending{username="john"}` — 使用者 "john" 的等待中作業數
- `slurm_account_jobs_pending{account="smith"}` — 帳號 "smith" 的等待中作業數

#### 節點指標

節點指標追蹤資源可用性、節點狀態及使用率。範例包含：

- `slurm_nodes` — 節點總數
- `slurm_nodes_idle` — 閒置（Idle）節點數量
- `slurm_nodes_alloc` — 已分配（Allocated）節點數量
- `slurm_node_cpus{node="nodename"}` — 指定節點的 CPU 數量
- `slurm_node_memory_bytes{node="nodename"}` — 指定節點的記憶體大小（位元組）

#### 分割區指標

分割區指標顯示各分割區的作業分佈及資源分配情況。範例包含：

- `slurm_partitions` — 分割區總數
- `slurm_partition_jobs{partition="name"}` — 指定分割區上的作業數
- `slurm_partition_nodes{partition="name"}` — 指定分割區上的節點數

以下指標可用於 [Slinky](https://slurm.schedmd.com/slinky.html) 或其他具有自動擴展（Auto-scale）功能的系統。透過了解分割區中作業請求的最大節點數，可考慮是否擴充該分割區的節點數量。被暫停的作業不計入這些指標。

- `slurm_partition_jobs_max_job_nodes_nohold` — 分割區中所有未暫停的等待作業裡，單一作業請求的最大節點數
- `slurm_partition_jobs_min_job_nodes_nohold` — 分割區中所有未暫停的等待作業裡，各作業最小節點需求數的最大值

#### 排程器指標

排程器指標提供排程效能與行為的洞察資訊。範例包含：

- `slurm_sched_cycle_cnt` — 排程循環（Scheduling cycle）計數
- `slurm_sched_cycle_last` — 最近一次排程循環的耗時
- `slurm_bf_cycle_cnt` — 回填（Backfill）循環計數
- `slurm_bf_active` — 回填排程器是否正在執行

---

### 安全性考量

指標系統有幾項重要的安全性含義：

- **無認證機制**：指標端點沒有內建的認證機制。任何能夠存取 slurmctld 連接埠的人均可查詢指標。

- **`PrivateData` 相依性**：當 `slurm.conf` 中設定了任何 `PrivateData` 參數時，指標功能將自動停用，以防止敏感資訊外洩。

- **網路存取控制**：指標透過 slurmctld 的網路介面對外公開。建議使用防火牆規則及網路分段（Network segmentation）來控制存取。

- **資訊揭露風險**：指標可能揭露叢集使用率、作業模式及使用者活動等資訊，在某些環境中可能被視為敏感資料。

---

### 效能影響

指標的收集與輸出可能影響 slurmctld 的效能：

- **鎖競爭（Lock contention）**：查詢指標需要取得 slurmctld 內部的各種鎖定，在高頻率查詢期間可能影響排程器效能。

- **資料收集開銷**：指標從 slurmctld 的內部資料結構即時收集，會增加運算負擔。

- **外部監控系統的資料處理開銷**：我們提供使用者和帳號指標等無界限實體的端點，監控系統可能將每個實體視為新的時間序列，這可能大幅增加儲存的資料量。

- **網路 I/O**：頻繁的指標查詢會產生網路流量，並消耗 slurmctld 的網路 I/O 容量，在擁有數千個作業、使用者或帳號的系統上尤為明顯。

降低效能影響的建議做法：

- 在監控系統中設定適當的抓取間隔（例如 60–120 秒）
- 盡可能在監控系統中使用快取機制
- 啟用指標功能時監控 slurmctld 的效能狀況
- 不要使用 `/metrics/jobs-users-accts` 等無界限指標端點將資料儲存至監控系統

---

### 使用範例

#### 基本 curl 範例

查詢作業指標：

```
$ curl http://slurmctld.example.com:6817/metrics/jobs
# HELP slurm_jobs Total number of jobs
# TYPE slurm_jobs gauge
slurm_jobs 42
# HELP slurm_jobs_running Number of jobs in Running state
# TYPE slurm_jobs_running gauge
slurm_jobs_running 15
# HELP slurm_jobs_pending Number of jobs in Pending state
# TYPE slurm_jobs_pending gauge
slurm_jobs_pending 27
...
```

查詢節點指標：

```
$ curl http://slurmctld.example.com:6817/metrics/nodes
# HELP slurm_nodes Total number of nodes
# TYPE slurm_nodes gauge
slurm_nodes 100
# HELP slurm_nodes_idle Number of nodes in Idle state
# TYPE slurm_nodes_idle gauge
slurm_nodes_idle 85
# HELP slurm_nodes_alloc Number of nodes in Allocated state
# TYPE slurm_nodes_alloc gauge
slurm_nodes_alloc 15
...
```

#### Prometheus 設定

在 **prometheus.yml** 中新增以下設定，使 Prometheus 抓取 Slurm 指標：

```yaml
scrape_configs:
  - job_name: 'slurm_jobs'
    static_configs:
      - targets: ['slurm.example.com:6817']
    metrics_path: '/metrics/jobs'

  - job_name: 'slurm_nodes'
    static_configs:
      - targets: ['slurm.example.com:6817']
    metrics_path: '/metrics/nodes'

  - job_name: 'slurm_partitions'
    static_configs:
      - targets: ['slurm.example.com:6817']
    metrics_path: '/metrics/partitions'

  - job_name: 'slurm_scheduler'
    static_configs:
      - targets: ['slurm.example.com:6817']
    metrics_path: '/metrics/scheduler'

  - job_name: 'slurm_useracct'
    static_configs:
      - targets: ['slurm.example.com:6817']
    metrics_path: '/metrics/jobs-users-accts'
```

---

### 相關文件

- [slurm.conf 設定參考](slurm.conf.html) — `MetricsType`、`PrivateData` 等參數說明
- [REST API](rest.md) — Slurm 的 REST API 存取方式
- [故障排除](troubleshoot.md) — 常見問題診斷
