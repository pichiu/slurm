# Slurm 排程設定指南 (Scheduling Configuration Guide)

## TL;DR

Slurm 在作業提交、完成等事件時執行快速簡單的排程嘗試。回填排程 (backfill) 是預設載入的外掛程式，會在不延遲高優先順序作業的前提下啟動低優先順序作業。關鍵設定包括 `SchedulerType`、`SchedulerParameters` 和各種 `bf_*` 參數。準確的作業時間限制對回填排程效能至關重要。

---

## 翻譯

### 概觀

Slurm 設計為在作業提交、完成和設定變更等事件時執行快速簡單的排程嘗試。在這些事件觸發的排程事件中，會考慮 `default_queue_depth`（預設 100）數量的作業。

在較不頻繁的間隔（由 `sched_interval` 定義），主排程迴圈會執行，考慮所有作業同時仍遵守 `partition_job_depth` 限制。

在這兩種情況下，作業都以嚴格的優先順序評估，一旦分割區中的任何作業或作業陣列任務保持等待中，該分割區中的其他作業將不會被排程，以避免佔用較高優先順序等待中作業所需的資源。

更全面的排程嘗試通常由回填排程外掛程式完成，它會考慮作業執行時間和所需資源，以確定較低優先順序的作業是否真的會佔用較高優先順序作業所需的資源。這允許回填排程器為等待中的作業分配更具體的[原因代碼](../users/job_reason_codes.md)，或啟動先前等待中的作業。

---

### 排程設定

**SchedulerType** 設定參數指定要使用的排程器外掛程式：

| 排程器 | 說明 |
|--------|------|
| `sched/backfill` | 執行回填排程（預設） |
| `sched/builtin` | 在每個分割區/佇列內以嚴格優先順序嘗試排程作業 |

**SchedulerParameters** 設定參數可以指定多種參數：

#### 通用排程參數

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `default_queue_depth=#` | 每個可能導致作業被排程的事件中要考慮的作業數 | 100 |
| `defer` | 不在提交時單獨嘗試排程作業。適用於高吞吐量運算 | 停用 |
| `max_switch_wait=#` | 作業等待所需葉交換器數量的最長時間（秒） | 300 |
| `partition_job_depth=#` | 在任何單一分割區中測試的作業數 | 0（無限制）|
| `sched_interval=#` | 主排程迴圈執行並測試所有等待作業的頻率（秒） | 60 |

---

### 回填排程

回填排程外掛程式預設載入。沒有回填排程，每個分割區會以嚴格優先順序排程，這通常會導致系統利用率和反應性顯著低於其他可能的情況。

回填排程會在不延遲**任何**較高優先順序作業的預期開始時間的情況下啟動較低優先順序的作業。由於等待作業的預期開始時間取決於執行中作業的預期完成時間，合理準確的時間限制對回填排程的良好運作非常重要。

#### 回填排程運作方式

1. Slurm 的回填排程器考慮每個執行中的作業
2. 然後以優先順序考慮等待中的作業，決定每個作業何時何地可以開始
3. 考慮因素包括：
   - [作業搶佔](preempt.md)
   - [Gang 排程](gang_scheduling.html)
   - [通用資源 (GRES) 需求](gres.md)
   - 記憶體需求等
4. 如果正在考慮的作業可以立即開始而不影響任何較高優先順序作業的預期開始時間，則啟動它
5. 否則，在作業預期執行期間保留所需資源
6. 回填外掛程式會設定等待作業的預期開始時間，將這些保留的節點設為「**Planned**」狀態

可使用 `squeue --start` 命令查看作業的預期開始時間。

**注意**：基於效能原因，回填排程器為作業保留整個節點，即使作業不需要整個節點。

#### 多分割區作業處理

排程邏輯建立作業-分割區對的排序清單。提交到多個分割區的作業在清單中有與請求分割區數量相同的項目。預設情況下，回填排程器可能會評估單一作業的所有作業-分割區對，可能為每對保留資源，但只在提供最早開始時間的保留中啟動作業。

#### 時間限制的重要性

沒有合理的作業時間限制估計，回填排程會很困難。以下設定參數可以幫助：

| 參數 | 說明 |
|------|------|
| `DefaultTime` | 預設作業時間限制（按分割區指定） |
| `MaxTime` | 最大作業時間限制（按分割區指定） |
| `OverTimeLimit` | 作業可以超過其時間限制的量，超過後會被終止（系統層級參數） |

---

### 回填排程參數

回填排程是一項耗時的操作。鎖會每兩秒短暫釋放，以便處理其他操作，例如處理新的作業提交請求。

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `bf_continue` | 如果設定，則在定期釋放鎖後繼續回填排程 | 停用 |
| `bf_interval=#` | 回填排程嘗試之間的間隔（秒） | 30 |
| `bf_max_job_part=#` | 每個回填週期中每個分割區啟動的最大作業數 | 0（無限制）|
| `bf_max_job_start=#` | 每個回填週期中啟動的最大作業數 | 0（無限制）|
| `bf_max_job_test=#` | 每個回填週期中考慮回填排程的最大作業數 | 100 |
| `bf_max_job_user=#` | 每個回填週期中每個使用者啟動的最大作業數 | 0（無限制）|
| `bf_max_time=#` | 回填排程器可花費的最長時間（秒），包括釋放鎖時的休眠時間 | bf_interval 值 |
| `bf_one_resv_per_job` | 不允許每個作業新增多個回填保留 | 停用 |
| `bf_resolution=#` | 回填排程的時間解析度（秒） | 60 |
| `bf_window=#` | 決定作業何時何地可以開始時向前查看的時間（分鐘） | 1440（一天）|
| `bf_yield_interval=#` | 釋放鎖的時間間隔（微秒） | 2,000,000（2 秒）|
| `bf_yield_sleep=#` | 釋放鎖的持續時間（微秒） | 500,000（0.5 秒）|

---

## 說明

### 排程類型比較

| 排程器 | 特點 | 適用場景 |
|--------|------|----------|
| `sched/builtin` | 嚴格優先順序，簡單 | 小型叢集，優先順序明確 |
| `sched/backfill` | 允許低優先作業填補空隙 | 大多數生產環境 |

### 回填排程的優缺點

**優點**：
- 提高系統利用率
- 允許小作業更快開始
- 減少資源閒置

**缺點**：
- 需要準確的時間限制
- 計算開銷較大
- 設定較複雜

### 時間限制與回填效能

```
時間限制準確度 ←──────────────────→ 回填效能
     低                             差
     高                             好
```

如果使用者傾向設定過長的時間限制，回填排程效能會下降，因為：
- 保留的時間窗口過大
- 較少的作業能「插隊」
- 資源利用率下降

---

## 實務範例

### 基本排程設定

```
# slurm.conf
SchedulerType=sched/backfill
SchedulerParameters=default_queue_depth=200,sched_interval=30
```

### 高吞吐量環境設定

```
# slurm.conf - 高吞吐量設定
SchedulerType=sched/backfill
SchedulerParameters=defer,bf_continue,bf_interval=10,bf_max_job_test=500
```

### 大型叢集設定

```
# slurm.conf - 大型叢集（>1000 節點）
SchedulerType=sched/backfill
SchedulerParameters=bf_interval=60,bf_max_job_test=1000,bf_resolution=120,bf_window=2880
```

### 分割區時間限制設定

```
# slurm.conf
PartitionName=debug  Nodes=node[01-10] MaxTime=30 DefaultTime=10
PartitionName=normal Nodes=node[11-100] MaxTime=1440 DefaultTime=60
PartitionName=long   Nodes=node[11-100] MaxTime=10080 DefaultTime=1440
```

### 監控排程效能

```bash
# 查看回填排程統計
sdiag

# 查看作業預期開始時間
squeue --start

# 查看等待原因
squeue -o "%.10i %.9P %.20j %.8u %.8T %.10M %.10l %R"
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 DefaultTime | 為每個分割區設定合理的預設時間限制 |
| bf_window 過小 | 設定至少與最長允許時間限制一樣長 |
| bf_max_job_test 過小 | 根據叢集大小和作業數調整 |
| 忽略時間限制準確性 | 教育使用者設定準確的時間限制 |
| bf_resolution 過小 | 增加解析度以減少開銷 |

### 效能調校建議

1. **監控 sdiag 輸出**：
   ```bash
   sdiag | grep -A 20 "Backfill"
   ```

2. **調整回填頻率**：
   - 小叢集：`bf_interval=30`
   - 大叢集：`bf_interval=60-120`

3. **平衡精確度與效能**：
   - 增加 `bf_resolution` 可減少開銷
   - 減少 `bf_window` 可加快處理

4. **高吞吐量設定**：
   - 使用 `defer` 減少提交時排程
   - 使用 `bf_continue` 持續排程

### 除錯技巧

```bash
# 查看排程器診斷資訊
sdiag

# 查看詳細排程日誌
# 在 slurm.conf 中設定：
# SlurmctldDebug=verbose
# DebugFlags=Backfill

# 查看作業為何等待
scontrol show job <jobid> | grep Reason
```

---

## 快速參考

### slurm.conf 排程設定

```
# 基本設定
SchedulerType=sched/backfill

# 排程參數
SchedulerParameters=default_queue_depth=100
SchedulerParameters+=sched_interval=60
SchedulerParameters+=partition_job_depth=0

# 回填參數
SchedulerParameters+=bf_interval=30
SchedulerParameters+=bf_max_job_test=100
SchedulerParameters+=bf_resolution=60
SchedulerParameters+=bf_window=1440
```

### 重要參數對照表

| 參數 | 預設值 | 建議範圍 |
|------|--------|----------|
| `default_queue_depth` | 100 | 50-500 |
| `sched_interval` | 60 秒 | 30-120 秒 |
| `bf_interval` | 30 秒 | 10-120 秒 |
| `bf_max_job_test` | 100 | 100-1000 |
| `bf_resolution` | 60 秒 | 60-300 秒 |
| `bf_window` | 1440 分鐘 | 720-2880 分鐘 |

### 診斷命令

| 命令 | 功能 |
|------|------|
| `sdiag` | 顯示排程統計 |
| `squeue --start` | 顯示作業預期開始時間 |
| `scontrol show config | grep Sched` | 顯示排程設定 |

### 相關文件

- [slurm.conf](slurm.conf.html) - 主要設定檔
- [搶佔](preempt.md) - 作業搶佔設定
- [Gang 排程](gang_scheduling.html) - Gang 排程設定
- [多因子優先順序](priority_multifactor.md) - 優先順序設定
