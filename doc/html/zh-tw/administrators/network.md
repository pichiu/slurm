# Slurm 網路設定指南

## TL;DR

Slurm 各元件預設連接埠：slurmctld（6817）、slurmd（6818）、slurmdbd（6819）、MySQL（3306）。使用者端命令連接 slurmctld 和 slurmdbd。srun 需額外設定 SrunPortRange。支援 IPv6（透過 CommunicationParameters=EnableIPv6）。多控制器和聯邦架構需開放額外通訊路徑。

---

## 翻譯

### 概觀

Slurm 叢集中有許多元件需要能夠相互通訊。某些站點有安全性需求，禁止開放機器之間的所有通訊，需要能夠選擇性地只開放必要的連接埠。本文件將說明不同元件之間需要的通訊配置。

下圖是一個相當典型的叢集配置，**slurmctld** 和 **slurmdbd** 在不同的機器上。在較小的叢集中，MySQL 可以與 **slurmdbd** 在同一台機器上運行，但在大多數情況下，最好讓它在專用機器上運行。**slurmd** 在計算節點上運行，使用者端命令可以安裝並從您選擇的機器上運行。

---

### slurmctld 的通訊

**slurm ctld** 監聽傳入請求的預設連接埠是 **6817**。此連接埠可透過 slurm.conf 的 [SlurmctldPort](slurm.conf.html#OPT_SlurmctldPort) 參數更改。slurmctld 在該連接埠監聽傳入請求，並在請求者開啟的同一連線上回應。

運行 **slurmctld** 的機器也需要能夠建立對外連線：
- 需要與 **slurmdbd** 通訊，預設連接埠 **6819**
- 需要與計算節點上的 **slurmd** 通訊，預設連接埠 **6818**

**IPv6 設定**：

預設情況下，**slurmctld** 監聽 IPv4 流量。可透過在 slurm.conf 的 [CommunicationParameters](slurm.conf.html#OPT_CommunicationParameters) 中新增 **EnableIPv6** 來啟用 IPv6 通訊。啟用 IPv6 後，可透過新增 **DisableIPv4** 來停用 IPv4。這些設定必須在 slurmdbd.conf 和 slurm.conf 中一致。

---

### slurmdbd 的通訊

**slurmdbd** 監聽傳入請求的預設連接埠是 **6819**。此連接埠可透過 slurmdbd.conf 的 [DbdPort](slurmdbd.conf.html#OPT_DbdPort) 參數更改。slurmdbd 在該連接埠監聽傳入請求，並在請求者開啟的同一連線上回應。

運行 **slurmdbd** 的機器需要能夠存取：
- MySQL 或 MariaDB 伺服器，預設連接埠 **3306**（可透過 [StoragePort](slurmdbd.conf.html#OPT_StoragePort) 更改）
- **slurmctld**，預設連接埠 **6817**

**IPv6 設定**：

與 slurmctld 相同，可透過 slurmdbd.conf 的 [CommunicationParameters](slurmdbd.conf.html#OPT_CommunicationParameters) 設定 IPv6。

---

### slurmd 的通訊

**slurmd** 監聽來自 **slurmctld** 傳入請求的預設連接埠是 **6818**。此連接埠可透過 slurm.conf 的 [SlurmdPort](slurm.conf.html#OPT_SlurmdPort) 參數更改。

**srun 連接埠範圍**：

運行 **srun** 的機器也使用一個連接埠範圍來與 **slurmstepd** 通訊。預設情況下，這些連接埠是從臨時連接埠範圍隨機選擇的，但您可以使用 [SrunPortRange](slurm.conf.html#OPT_SrunPortRange) 指定可選擇的連接埠範圍。對於在防火牆後面的登入節點，這是必要的設定。

運行 **slurmd** 的機器需要能夠與 **slurmctld** 建立連線，預設連接埠 **6817**。

---

### 使用者端命令的通訊

大多數使用者端命令會與 **slurmctld** 通訊，預設連接埠 **6817**。這包括以下命令：

| 命令 | 說明 |
|------|------|
| salloc | 分配資源 |
| sacctmgr | 帳戶管理 |
| sbatch | 提交批次作業 |
| sbcast | 廣播檔案 |
| scancel | 取消作業 |
| scontrol | 控制命令 |
| sdiag | 診斷資訊 |
| sinfo | 節點資訊 |
| sprio | 優先順序資訊 |
| squeue | 佇列資訊 |
| sshare | 分享資訊 |
| sstat | 作業統計 |
| strigger | 觸發器管理 |
| sview | 圖形介面 |

以下命令直接與 **slurmdbd** 通訊，預設連接埠 **6819**：

| 命令 | 說明 |
|------|------|
| sacct | 計費查詢 |
| sacctmgr | 帳戶管理 |
| sreport | 報表 |

---

### srun 通訊流程

當使用者使用 **srun** 啟動作業時，必須有從呼叫 **srun** 的機器到作業分配節點的通訊路徑。通訊按以下順序進行：

1. srun 發送作業分配請求到 slurmctld
2. slurmctld 授予分配並回傳詳細資訊
3. srun 發送步驟建立請求到 slurmctld
4. slurmctld 回應步驟憑證
5. srun 開啟 I/O socket
6. srun 將憑證與任務資訊轉發到 slurmd
7. slurmd 按需轉發請求（依扇出）
8. slurmd fork/exec slurmstepd
9. slurmstepd 連接 I/O 並啟動任務
10. 任務終止時，slurmstepd 通知 srun
11. srun 通知 slurmctld 作業終止
12. slurmctld 透過 slurmd 驗證所有程序終止並釋放資源

---

### 多控制器通訊

您可以設定次要 **slurmctld** 和/或 **slurmdbd** 作為主要服務故障時的備援。連接埠不變，但需要考慮額外的通訊路徑：

- 使用者端命令需要能夠存取兩台運行 **slurmctld** 的機器
- 使用者端命令需要能夠存取兩台運行 **slurmdbd** 的機器
- 兩個 **slurmctld** 實例都需要能夠存取兩個 **slurmdbd** 實例
- 每個 **slurmdbd** 都需要能夠存取 MySQL 伺服器

---

### 多叢集通訊

在多個 **slurmctld** 實例共用同一個 **slurmdbd** 的環境中，您可以將每個叢集設定為獨立運行，並允許使用者指定要提交作業的叢集。所有 **slurmctld** 實例都需要能夠與同一個 **slurmdbd** 實例通訊。

更多關於多叢集配置的資訊，請參閱[多叢集操作](multi_cluster.html)文件。

---

### 聯邦通訊

Slurm 還提供在多個叢集之間以點對點方式排程作業的能力，允許作業在首先有可用資源的叢集上運行。與多叢集配置的通訊需求差異在於，兩個 **slurmctld** 實例需要能夠相互通訊。

更多關於使用聯邦的詳細資訊，請參閱[聯邦](federation.md)文件。

---

### IPv6 通訊

**slurmctld**、**slurmdbd** 和 **slurmd** 守護程式預設使用 IPv4 通訊，但可以設定為使用 IPv6。

**啟用方式**：

在 slurm.conf 和 slurmdbd.conf 中設定：
```
CommunicationParameters=EnableIPv6
```

然後重新啟動所有守護程式。**slurmd** 在此模式下可以透過 IPv4 或 IPv6 運作。

**停用 IPv4**：
```
CommunicationParameters=EnableIPv6,DisableIPv4
```

在此模式下，所有設備都必須有有效的 IPv6 位址，否則連線將失敗。

**注意事項**：

- **slurmctld** 預期節點對應到單一 IP 位址（**getaddrinfo()** 查詢時回傳的第一個位址）
- 如果在現有叢集上啟用 IPv6 且節點有 IPv6 位址，必須重新啟動 **slurmd** 守護程式才能建立 IPv6 通訊
- /etc/gai.conf 中的 `precedence ::ffff:0:0/96 100` 設定會導致 IPv4 位址在 IPv6 位址之前回傳
- 可以呼叫 `scontrol setdebugflags +NET` 在 slurmctld.log 中啟用網路相關除錯記錄

---

## 說明

### 連接埠對照表

| 服務 | 預設連接埠 | 設定參數 |
|------|-----------|----------|
| slurmctld | 6817 | SlurmctldPort |
| slurmd | 6818 | SlurmdPort |
| slurmdbd | 6819 | DbdPort |
| MySQL/MariaDB | 3306 | StoragePort |

### 通訊架構圖

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   使用者端    │────▶│  slurmctld   │────▶│   slurmd     │
│  (命令列)    │     │   (6817)     │     │   (6818)     │
└──────────────┘     └──────┬───────┘     └──────────────┘
                           │
                           ▼
                    ┌──────────────┐     ┌──────────────┐
                    │  slurmdbd    │────▶│    MySQL     │
                    │   (6819)     │     │   (3306)     │
                    └──────────────┘     └──────────────┘
```

---

## 實務範例

### 基本防火牆設定

```bash
# 允許 slurmctld 連接埠
firewall-cmd --permanent --add-port=6817/tcp

# 允許 slurmd 連接埠
firewall-cmd --permanent --add-port=6818/tcp

# 允許 slurmdbd 連接埠
firewall-cmd --permanent --add-port=6819/tcp

# 重新載入防火牆
firewall-cmd --reload
```

### srun 連接埠範圍設定

```
# slurm.conf
SrunPortRange=60001-63000
```

```bash
# 防火牆設定
firewall-cmd --permanent --add-port=60001-63000/tcp
firewall-cmd --reload
```

### IPv6 設定

```
# slurm.conf
CommunicationParameters=EnableIPv6

# slurmdbd.conf
CommunicationParameters=EnableIPv6
```

### 多控制器防火牆規則

```bash
# 在兩台 slurmctld 機器上
firewall-cmd --permanent --add-port=6817/tcp
firewall-cmd --permanent --add-port=6819/tcp

# 在兩台 slurmdbd 機器上
firewall-cmd --permanent --add-port=6819/tcp
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未開放必要連接埠 | 確認所有 Slurm 連接埠已開放 |
| srun 無法連接計算節點 | 設定 SrunPortRange 並開放對應連接埠 |
| IPv6 設定不一致 | slurm.conf 和 slurmdbd.conf 設定需一致 |
| 忽略 slurmdbd 到 slurmctld 的連線 | slurmdbd 也需要能連接 slurmctld |

### 建議

1. **最小化開放**：
   - 只開放必要的連接埠
   - 使用防火牆限制來源 IP

2. **srun 環境**：
   - 登入節點在防火牆後時務必設定 SrunPortRange
   - 範圍大小根據預期同時執行的作業數決定

3. **IPv6 遷移**：
   - 先在測試環境驗證
   - 確認所有節點都有有效的 IPv6 位址
   - 使用 `scontrol setdebugflags +NET` 進行除錯

---

## 快速參考

### 預設連接埠

| 服務 | 連接埠 | 方向 |
|------|--------|------|
| slurmctld | 6817 | 入站 |
| slurmd | 6818 | 入站 |
| slurmdbd | 6819 | 入站 |
| MySQL | 3306 | 出站（從 slurmdbd）|

### slurm.conf 網路設定

```
SlurmctldPort=6817
SlurmdPort=6818
SrunPortRange=60001-63000
CommunicationParameters=EnableIPv6  # 可選
```

### slurmdbd.conf 網路設定

```
DbdPort=6819
StoragePort=3306
CommunicationParameters=EnableIPv6  # 可選
```

### 相關文件

- [聯邦排程](federation.md) - 聯邦通訊設定
- [故障排除](troubleshoot.md) - 網路問題診斷
- [大型叢集管理](big_sys.md) - 大規模網路考量

