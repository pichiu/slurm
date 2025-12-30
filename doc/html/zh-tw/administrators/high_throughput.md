# Slurm 高吞吐量運算管理指南 (High Throughput Computing Administration Guide)

## TL;DR

高吞吐量運算 (HTC) 涉及執行大量短作業。Slurm 已驗證可持續每秒執行 500 個簡單批次作業。關鍵調校：系統層級（file-max、tcp_max_syn_backlog、somaxconn）、Munge 執行緒數、Slurm 設定（defer、MinJobAge、bf_interval）。考慮停用 lua job submit plugin、減少日誌層級、使用 CommitDelay 加速 SlurmDBD。

---

## 翻譯

### 概觀

本文件包含針對高吞吐量運算的 Slurm 管理員資訊，即執行許多短作業的情況。要獲得高吞吐量運算的最佳效能需要一些調校，本文件將幫助您有個好的開始。閱讀本資料前，建議先具備 Slurm 的基本知識。

### 效能結果

Slurm 已驗證可在持續基礎上每秒執行 500 個簡單批次作業，並在短時間內爆發達到更高水準。實際效能取決於要執行的作業以及使用的硬體和設定。

---

### 系統設定

多個系統設定參數可能需要修改以支援大量開啟檔案和 TCP 連接的大量訊息爆發。可透過 `/etc/rc.d/rc.local` 或 `/etc/sysctl.conf` 腳本進行變更以在重新開機後保留設定。

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `/proc/sys/fs/file-max` | 最大同時開啟檔案數 | 至少 32,832 |
| `/proc/sys/net/ipv4/tcp_max_syn_backlog` | 記憶體中 SYN 請求的最大數量 | 視負載增加 |
| `/proc/sys/net/ipv4/tcp_syncookies` | 啟用 syncookies | 1 |
| `/proc/sys/net/ipv4/tcp_synack_retries` | SYN,ACK 重傳次數 | 預設 5 通常足夠 |
| `/proc/sys/net/core/somaxconn` | socket listen() backlog 限制 | 1024+ |
| `/proc/sys/net/ipv4/ip_local_port_range` | 臨時埠範圍 | 32768 65535 |

**設定範例**：
```bash
# /etc/sysctl.conf
fs.file-max = 32832
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_syncookies = 1
net.core.somaxconn = 1024
net.ipv4.ip_local_port_range = 32768 65535
```

**網路佇列長度**：
```bash
ifconfig <interface> txqueuelen 4096
```

---

### Munge 設定

預設 Munge 守護程式以兩個執行緒執行，但較高的執行緒數可提高吞吐量。

```bash
# 建議使用 10 個執行緒
munged --num-threads 10
```

---

### 使用者限制

slurmctld 守護程式的 ulimit 值應設定較高，包括：
- 記憶體大小
- 開啟檔案數
- 堆疊大小

---

### Slurm 設定

#### 計費與日誌設定

| 參數 | 建議 | 說明 |
|------|------|------|
| AccountingStorageType | 可選擇停用 | 停用可略微提升效能；使用 SlurmDBD 時設定 CommitDelay |
| JobAcctGatherType | `jobacct_gather/none` | 停用作業計費資訊收集 |
| JobCompType | `jobcomp/none` | 停用作業完成資訊記錄 |
| SlurmctldDebug | `error` 或 `info` | 較詳細的日誌會降低吞吐量 |
| SlurmdDebug | `error` 或 `info` | 較詳細的日誌會降低吞吐量 |
| SlurmdLogFile | 本地儲存 | 建議寫入本地儲存 |

#### 不建議用於高吞吐量的設定

| 設定 | 原因 |
|------|------|
| JobSubmitPlugins (lua) | slurmctld 在持有內部鎖時執行此腳本，同時只能執行一個副本，嚴重限制並行性 |
| PrologSlurmctld/EpilogSlurmctld | 每個作業啟動都需建立獨立執行緒並獲取寫入鎖，嚴重限制排程器吞吐量 |

#### 作業管理設定

| 參數 | 說明 | 預設值 |
|------|------|--------|
| MaxJobCount | slurmctld 可記錄的最大作業數 | 10,000 |
| MessageTimeout | 等待回應的時間 | 10 秒 |
| MinJobAge | 已完成作業可從記憶體清除的最短時間 | 300 秒（建議減少）|
| PriorityType | `priority/basic` 最快（僅 FIFO）| multifactor |

#### 排程參數

| 參數 | 說明 | 建議值 |
|------|------|--------|
| batch_sched_delay | 批次作業排程可延遲的時間 | 3 秒（預設）|
| defer | 延遲個別作業排程 | 適用於大量同時提交 |
| defer_batch | 類似 defer，但允許互動作業立即啟動 | - |
| sched_min_interval | 排程邏輯執行頻率（微秒）| 2000000 |
| default_queue_depth | 每次排程迭代考慮的作業數 | 100 |

#### 回填排程參數

| 參數 | 說明 | 建議值 |
|------|------|--------|
| bf_max_job_test | 每個回填週期考慮的最大作業數 | 100 或更少 |
| bf_interval | 回填間隔 | 30 秒或更長 |
| bf_max_job_user | 每使用者最大作業數 | 視情況 |
| bf_resolution | 回填解析度 | 視情況 |
| bf_window | 回填視窗 | 視情況 |

#### 高吞吐量範例設定

以下設定用於在某叢集上持續每秒執行數百個作業：

```
# slurm.conf - 高吞吐量設定範例
SchedulerParameters=batch_sched_delay=20
SchedulerParameters+=bf_continue
SchedulerParameters+=bf_interval=300
SchedulerParameters+=bf_min_age_reserve=10800
SchedulerParameters+=bf_resolution=600
SchedulerParameters+=bf_yield_interval=1000000
SchedulerParameters+=partition_job_depth=500
SchedulerParameters+=sched_max_job_start=200
SchedulerParameters+=sched_min_interval=2000000
```

---

### SlurmctldParameters 設定

#### conmgr_max_connections

增加此值允許 slurmctld 同時接受更多連接以避免 connect() 逾時。

**權衡**：
- 更多連接 = 更多記憶體使用
- 應至少為硬體 CPU 執行緒數
- 應小於 `net.nf_conntrack_max` 和 `net.core.somaxconn`
- 過高可能導致記憶體不足崩潰

**建議**：先嘗試調整 MessageTimeout 和 TCPTimeout。

#### conmgr_threads

控制處理通訊的執行緒池大小。

**權衡**：
- 更多執行緒 = 更多記憶體使用
- 執行緒多於 CPU 會增加核心排程器競爭
- 獲取鎖會與排程器執行緒競爭

**建議**：大多數情況下應考慮減少而非增加執行緒數。

#### RPC 速率限制（23.02+）

基於每使用者的虛擬令牌桶機制：

| 參數 | 說明 |
|------|------|
| rl_enable | 啟用速率限制 |
| rl_bucket_size | 最大令牌數 |
| rl_refill_rate | 令牌補充速率 |
| rl_refill_period | 令牌補充頻率 |
| rl_table_size | 追蹤的實體數 |

---

### SlurmctldPort 設定

建議設定 slurmctld 接受多個埠的連入訊息，以避免超過 SOMAXCONN 限制。建議使用 2-10 個埠。

```
# slurm.conf
SlurmctldPort=6817-6826
```

---

### SlurmDBD 設定

#### CommitDelay

設定 slurmdbd.conf 中的 CommitDelay 選項，在 slurmdbd 接收連接和提交資料庫之間引入延遲，允許多個請求累積並減少提交次數。

#### 資料保留

高作業吞吐量會導致資料庫快速增長。建議：

1. 提前規劃資料保留策略
2. 設定相關的 Purge 選項
3. 考慮清除頻率
4. 或使用 AccountingStorageEnforce 的 `nosteps` 和 `nojobs` 選項跳過儲存

---

## 說明

### 高吞吐量 vs 高效能運算

| 特性 | 高吞吐量 (HTC) | 高效能 (HPC) |
|------|----------------|--------------|
| 作業數量 | 大量 | 較少 |
| 作業長度 | 短 | 長 |
| 優先順序 | 可能 FIFO | 通常多因子 |
| 計費需求 | 可選 | 通常需要 |
| 回填重要性 | 較低 | 較高 |

### 瓶頸分析

```
作業提交 → slurmctld 處理 → 排程 → 作業啟動
    │           │           │         │
    └─ 網路     └─ 鎖競爭   └─ 演算法  └─ slurmd
```

高吞吐量環境中的潛在瓶頸：
1. 網路連接限制
2. slurmctld 鎖競爭
3. 排程演算法開銷
4. 資料庫寫入

---

## 實務範例

### 系統層級設定

```bash
# /etc/sysctl.conf
fs.file-max = 65536
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_syncookies = 1
net.core.somaxconn = 2048
net.ipv4.ip_local_port_range = 32768 65535

# 套用設定
sysctl -p
```

### Munge 高吞吐量設定

```bash
# /etc/sysconfig/munge (或 systemd override)
OPTIONS="--num-threads 10"
```

### 完整 slurm.conf 高吞吐量設定

```
# slurm.conf - 高吞吐量設定

# 停用不必要的功能
JobAcctGatherType=jobacct_gather/none
JobCompType=jobcomp/none

# 減少日誌
SlurmctldDebug=info
SlurmdDebug=info

# 優先順序設定
PriorityType=priority/basic    # 或繼續使用 multifactor

# 作業管理
MaxJobCount=50000
MinJobAge=60
MessageTimeout=30

# 排程參數
SchedulerType=sched/backfill   # 或 sched/builtin
SchedulerParameters=batch_sched_delay=20
SchedulerParameters+=defer_batch
SchedulerParameters+=sched_min_interval=2000000
SchedulerParameters+=bf_interval=300
SchedulerParameters+=bf_max_job_test=100
SchedulerParameters+=bf_resolution=600
SchedulerParameters+=partition_job_depth=500

# 多埠
SlurmctldPort=6817-6820
```

### slurmdbd.conf 高吞吐量設定

```
# slurmdbd.conf
CommitDelay=5

# 資料保留
PurgeEventAfter=1month
PurgeJobAfter=1month
PurgeStepAfter=1month
```

### 監控高吞吐量效能

```bash
# 查看排程統計
sdiag

# 關注項目
sdiag | grep -A 20 "Main schedule statistics"
sdiag | grep -A 20 "Remote Procedure Call"

# 監控作業率
watch -n 5 'squeue | wc -l'
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 使用 lua job submit plugin | 避免使用或簡化邏輯 |
| MinJobAge 過長 | 減少到數秒 |
| 過多 slurmctld 日誌 | 設為 info 或 error |
| 未設定 CommitDelay | 使用 SlurmDBD 時設定此選項 |
| 系統限制未調整 | 調整 file-max、somaxconn 等 |
| conmgr_threads 過高 | 大多數情況應減少 |

### 調校步驟

1. **基準測試**：
   ```bash
   # 測量當前吞吐量
   time for i in $(seq 1 1000); do sbatch -o /dev/null test.sh; done
   ```

2. **識別瓶頸**：
   ```bash
   sdiag | grep -E "ave_time|cycle"
   ```

3. **逐步調整**：
   - 先調整系統參數
   - 再調整 Slurm 設定
   - 最後微調排程參數

4. **驗證效果**：
   - 重複基準測試
   - 比較 sdiag 輸出

---

## 快速參考

### 系統設定

```bash
# /etc/sysctl.conf
fs.file-max = 32832
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_syncookies = 1
net.core.somaxconn = 1024
net.ipv4.ip_local_port_range = 32768 65535
```

### slurm.conf 設定

```
# 高吞吐量基本設定
JobAcctGatherType=jobacct_gather/none
MinJobAge=60
SlurmctldDebug=info
SchedulerParameters=defer_batch,sched_min_interval=2000000
```

### 監控命令

| 命令 | 功能 |
|------|------|
| `sdiag` | 排程診斷資訊 |
| `squeue \| wc -l` | 佇列中作業數 |
| `scontrol show config \| grep -i sched` | 排程設定 |

### 相關文件

- [排程設定](sched_config.md) - 排程器設定
- [大型叢集管理](big_sys.md) - 大型系統設定
- [計費](accounting.md) - 計費設定
- [SlurmDBD](slurmdbd.html) - 資料庫守護程式設定
