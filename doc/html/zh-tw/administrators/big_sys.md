# Slurm 大型叢集管理指南 (Large Cluster Administration Guide)

## TL;DR

本文件針對 1,024 節點以上的叢集提供 Slurm 管理員指引。關鍵最佳化：調整系統參數（file-max、tcp_max_syn_backlog、somaxconn）、使用 `select/linear` 取代 `select/cons_tres`、減少作業計費開銷、設定較長的 `SlurmdTimeout`（120 秒以上）、調整 `TreeWidth` 為節點數的立方根。大型系統效能：30,000 任務在 15,000 節點上約 30 秒。

---

## 翻譯

### 概觀

本文件包含專門針對 1,024 節點或更多的叢集的 Slurm 管理員資訊。

**目前由 Slurm 管理的大型系統範例**：

| 系統 | 機構 | 核心數 |
|------|------|--------|
| Frontier | 橡樹嶺國家實驗室 (ORNL) | 8,699,904 |
| 天河二號 | 中國國防科技大學 | 4,981,760 |
| Perlmutter | 國家能源研究科學計算中心 (NERSC) | 761,856 |

Slurm 在更大數量級系統上的運作已透過模擬驗證。在該規模獲得最佳效能確實需要一些調校，本文件應能幫助您有個好的開始。

**先決條件**：應具備 Slurm 的工作知識。

---

### 效能

以下時間為執行列印 "Hello world" 並退出的 MPI 程式，包括處理輸出的時間。您的效能可能因硬體、軟體和設定差異而有所不同。

| 規模 | 節點數 | 系統 | 時間 |
|------|--------|------|------|
| 1,966,080 任務 | 122,880 | BlueGene/Q | 322 秒 |
| 30,000 任務 | 15,000 | Linux 叢集 | 30 秒 |

---

### 系統設定

必須設定三個系統參數以支援大量開啟的檔案和具有大量訊息突發的 TCP 連線。

可使用 `/etc/rc.d/rc.local` 或 `/etc/sysctl.conf` 腳本進行變更，以在重新開機後保留變更。

#### 必要參數

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `/proc/sys/fs/file-max` | 同時開啟檔案的最大數量 | 至少 388067 或作業系統預設值（取較大者）|
| `/proc/sys/net/ipv4/tcp_max_syn_backlog` | 尚未收到確認的已記憶連線請求最大數量 | 如果伺服器過載，嘗試增加 |
| `/proc/sys/net/core/somaxconn` | socket listen() 積壓限制 | 設定為支援突發請求數（如 1024）|

**設定範例**：
```bash
echo 388067 > /proc/sys/fs/file-max
echo 4096 > /proc/sys/net/ipv4/tcp_max_syn_backlog
echo 1024 > /proc/sys/net/core/somaxconn
```

#### 傳輸佇列長度

傳輸佇列長度 (txqueuelen) 也可能需要修改：

```bash
ifconfig <interface> txqueuelen 4096
```

---

### 執行緒/程序限制

SLES 12 SP2（用於 CLE 6.0UP04 的 Cray 系統）中引入了新限制。systemd 限制每個 init 腳本或 systemd 服務預設為 512 個執行緒/程序。

**解決方案**：

**使用 systemd 服務檔案**：
```ini
[Service]
TasksMax=infinity
```

**使用 init 腳本**：

建立 `/etc/systemd/system/<init script name>.service.d/override.conf`：
```ini
[Service]
TasksMax=infinity
```

**注意**：較早版本的 systemd 如果不支援 PIDs cgroup 控制器，會忽略 TasksMax 設定。

---

### 使用者限制

slurmctld 守護程式生效的 **ulimit** 值應設定較高：
- 記憶體大小
- 開啟檔案數
- 堆疊大小

---

### 節點選擇外掛程式 (SelectType)

| 外掛程式 | 說明 | 建議 |
|----------|------|------|
| `select/linear` | 分配整個節點 | **大型叢集建議** |
| `select/cons_tres` | 分配節點內的個別處理器和記憶體 | 較小叢集適用，有額外開銷 |

**建議**：為獲得最佳可擴展性，使用 `select/linear` 分配整個節點。

---

### 作業計費收集外掛程式 (JobAcctGatherType)

作業計費依賴每個運算節點上的 slurmstepd 守護程式定期取樣資料。此資料收集會消耗運算週期，產生所謂的「系統雜訊」。

| 選項 | 說明 |
|------|------|
| `jobacct_gather/none` | **最佳應用程式效能**（停用作業計費）|
| `JobAcctGatherFrequency=task=300` | 如果需要計費，設定較大的取樣間隔 |
| `JobCompType` | 考慮使用作業完成記錄，開銷較小 |

---

### 節點設定

明確指定預期設定以最佳化效能：

```
NodeName=node[001-1000] CPUs=32 RealMemory=64000 TmpDisk=100000
```

**建議**：
- 使用最少的 slurm.conf 行數設定節點
- 明確指定 `RealMemory`、`CPUs` 和 `TmpDisk`
- 如果節點資源少於設定，會被標記為 DOWN

---

### 計時器設定

#### EioTimeout

控制 srun 命令等待 slurmstepd 關閉 TCP/IP 連線的時間。

| 設定 | 預設值 | 建議 |
|------|--------|------|
| `EioTimeout` | 60 秒 | 較大系統和/或較慢網路可能需要更高值 |

#### MinJobAge

已終止作業保留在控制器中的最短時間。

| 設定 | 建議 |
|------|------|
| `MinJobAge` | 高吞吐量作業設定為最小實際值 |

#### SlurmdTimeout

slurmctld 與 slurmd 通訊的間隔（實際通訊在 SlurmdTimeout/2）。

| 叢集規模 | 建議值 |
|----------|--------|
| 非常大型叢集 | 120 秒或更多 |

#### EpilogMsgTime

當分配大量節點的作業完成時，可能同時發送大量訊息。使用此參數將訊息流量分散。

---

### 其他設定

#### TreeWidth

Slurm 使用階層式通訊。TreeWidth 控制訊息的扇出。

| 設定 | 預設值 | 說明 |
|------|--------|------|
| `TreeWidth` | 16 | 每個 slurmd 最多與 16 個其他 slurmd 通訊 |

**最佳化建議**：設定為叢集節點數的立方根。

| 節點數 | 建議 TreeWidth |
|--------|----------------|
| 1,000 | 10 |
| 8,000 | 20 |
| 27,000 | 30 |
| 125,000 | 50 |

#### 開啟檔案限制

srun 命令會自動增加開啟檔案限制到硬限制。

**建議**：在整個叢集設定開啟檔案硬限制為 8192。

---

## 說明

### 可擴展性瓶頸

| 瓶頸 | 解決方案 |
|------|----------|
| 檔案描述符耗盡 | 增加 file-max |
| TCP 連線積壓 | 增加 tcp_max_syn_backlog 和 somaxconn |
| 節點選擇開銷 | 使用 select/linear |
| 作業計費開銷 | 減少或停用計費 |
| 通訊延遲 | 調整 TreeWidth |

### 大型叢集最佳化層級

```
第 1 層：系統參數
└── file-max, somaxconn, tcp_max_syn_backlog

第 2 層：Slurm 設定
└── SelectType, JobAcctGatherType, TreeWidth

第 3 層：計時器調校
└── SlurmdTimeout, EioTimeout, MinJobAge

第 4 層：節點設定
└── 明確指定資源，最少設定行數
```

---

## 實務範例

### 大型叢集 slurm.conf 範本

```
# 叢集基本設定
ClusterName=large_cluster
SlurmctldHost=controller1
SlurmctldHost=controller2

# 最佳化選擇
SelectType=select/linear
SelectTypeParameters=CR_CPU

# 計費最佳化
JobAcctGatherType=jobacct_gather/none
JobCompType=jobcomp/mysql
JobCompLoc=slurm_acct_db

# 計時器最佳化
SlurmdTimeout=180
EioTimeout=120
MinJobAge=300
EpilogMsgTime=5000

# 階層通訊（50,000 節點叢集）
TreeWidth=37

# 節點設定
NodeName=node[00001-50000] CPUs=64 RealMemory=256000 TmpDisk=500000 State=UNKNOWN

# 分割區
PartitionName=batch Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

### 系統參數設定腳本

```bash
#!/bin/bash
# /etc/rc.local 或 /etc/sysctl.conf 補充

# 檔案描述符
echo 500000 > /proc/sys/fs/file-max

# TCP 連線
echo 8192 > /proc/sys/net/ipv4/tcp_max_syn_backlog
echo 4096 > /proc/sys/net/core/somaxconn

# 網路介面
for iface in eth0 ib0; do
    if [ -d /sys/class/net/$iface ]; then
        ifconfig $iface txqueuelen 4096
    fi
done
```

### sysctl.conf 設定

```
# /etc/sysctl.conf
fs.file-max = 500000
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 4096
```

### systemd 服務覆蓋

```bash
# 建立覆蓋目錄
mkdir -p /etc/systemd/system/slurmctld.service.d/
mkdir -p /etc/systemd/system/slurmd.service.d/

# slurmctld 覆蓋
cat > /etc/systemd/system/slurmctld.service.d/override.conf << EOF
[Service]
TasksMax=infinity
LimitNOFILE=500000
LimitMEMLOCK=infinity
EOF

# slurmd 覆蓋
cat > /etc/systemd/system/slurmd.service.d/override.conf << EOF
[Service]
TasksMax=infinity
LimitNOFILE=500000
EOF

# 重新載入 systemd
systemctl daemon-reload
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 使用 select/cons_tres | 大型叢集使用 select/linear |
| 預設系統參數 | 調高 file-max、somaxconn 等 |
| 過短的 SlurmdTimeout | 設定 120 秒或更長 |
| 啟用詳細作業計費 | 使用 jobacct_gather/none 或大取樣間隔 |
| 預設 TreeWidth | 設定為節點數的立方根 |
| 未設定 ulimit | 提高 slurmctld 的 ulimit |

### 效能監控

```bash
# 監控 slurmctld 效能
sdiag

# 檢查排程效能
squeue --start | head -20

# 查看 RPC 統計
scontrol show statistics

# 監控系統資源
lsof -u slurm | wc -l  # 開啟檔案數
ss -s  # TCP 連線統計
```

### 疑難排解

```bash
# 檢查系統參數
cat /proc/sys/fs/file-max
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
cat /proc/sys/net/core/somaxconn

# 檢查 ulimit
su - slurm -c "ulimit -a"

# 檢查 systemd 限制
systemctl show slurmctld | grep -i task

# 監控通訊
netstat -an | grep 6817 | wc -l
```

---

## 快速參考

### 系統參數

| 參數 | 建議值 |
|------|--------|
| file-max | 388067+ |
| tcp_max_syn_backlog | 4096+ |
| somaxconn | 1024+ |
| txqueuelen | 4096 |

### slurm.conf 設定

| 參數 | 大型叢集建議 |
|------|--------------|
| SelectType | select/linear |
| JobAcctGatherType | jobacct_gather/none |
| SlurmdTimeout | 120+ 秒 |
| EioTimeout | 120 秒 |
| MinJobAge | 300 秒 |
| TreeWidth | 節點數立方根 |

### TreeWidth 速查表

| 節點數 | TreeWidth |
|--------|-----------|
| 1,000 | 10 |
| 5,000 | 17 |
| 10,000 | 22 |
| 50,000 | 37 |
| 100,000 | 47 |

### 相關文件

- [slurm.conf](slurm.conf.html) - 主要設定檔
- [排程設定](sched_config.md) - 排程器調校
- [高吞吐量計算](high_throughput.html) - 高吞吐量設定
- [疑難排解](troubleshoot.md) - 問題排解
