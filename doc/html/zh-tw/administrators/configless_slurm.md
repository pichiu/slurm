# Slurm 無設定檔模式 (Configless Slurm)

## TL;DR

「無設定檔」Slurm 允許運算節點的 slurmd 和登入節點的使用者命令直接從 slurmctld 取得設定資訊，而非從本地預先部署的檔案。透過 `SlurmctldParameters=enable_configless` 啟用，slurmd 可使用 `--conf-server` 選項或 DNS SRV 記錄連接控制器。登入節點可使用 sackd 管理設定檔以減少網路請求。

---

## 翻譯

### 概觀

「無設定檔」Slurm 是一項功能，允許運算節點（特別是 slurmd 程序）和登入節點上執行的使用者命令直接從 slurmctld 取得設定資訊，而非從預先部署的本地檔案。

您的叢集仍需要在 Slurm 控制器上有一組中央設定檔 — Slurm 所謂的「無設定檔」意指運算節點、登入節點和其他叢集主機不需要部署這些檔案的本地副本。

slurmd 啟動時會連接您指定的 slurmctld，設定檔將被拉取到節點。此 slurmctld 可透過明確選項或（更佳地）透過叢集內定義的 DNS SRV 記錄來識別。

如果您有登入節點執行客戶端命令，這些客戶端命令在執行時必須使用 DNS 記錄從控制器取得設定資訊。如果預期登入節點有大量流量，這可能產生大量設定檔請求。在這種情況下，可使用 sackd 管理節點的設定檔以減少網路請求。

---

### 安裝

從 Slurm 20.02 開始，此功能已內建，無需額外安裝步驟。

---

### 設定

#### 控制器設定

在 slurm.conf 中設定並重新啟動 slurmctld：

```
SlurmctldParameters=enable_configless
```

#### slurmd 設定

有兩種方式設定 slurmd 從 slurmctld 取得設定：

**方式一：命令列選項**

```bash
slurmd --conf-server slurmctl-primary:6817
```

指定連接埠是選用的，預設為 6817。

多個 slurmctld 可以逗號分隔清單指定，按優先順序排列（最高到最低）：

```bash
slurmd --conf-server slurmctl-primary:6817,slurmctl-secondary
```

**方式二：DNS SRV 記錄**

確保節點上沒有本地設定檔，並設定 DNS SRV 記錄：

```
_slurmctld._tcp 3600 IN SRV 10 0 6817 slurmctl-backup
_slurmctld._tcp 3600 IN SRV 0 0 6817 slurmctl-primary
```

**注意**：
- `--conf-server` 選項優先於 DNS 記錄
- DNS SRV 記錄中，優先順序值最低的應為主要 slurmctld
- 如果部署 HA 設定，可指定多個 SRV 記錄

---

### 初始測試

設定 slurmctld 並啟動 slurmd 後，檢查設定是否存在於節點上：

1. 設定檔位於 **SlurmdSpoolDir** 下的 `/conf-cache/`
2. 會自動在 `/run/slurm/conf` 建立符號連結

**測試重新載入：**

1. 在 slurmctld 節點的 slurm.conf 中新增註解
2. 執行 `scontrol reconfig`
3. 檢查設定是否已更新

---

### 限制

| 限制項目 | 說明 |
|----------|------|
| SlurmdSpoolDir/SlurmdPidFile 的 %n | 除非 slurmd 同時使用 `-N` 選項，否則無法正確替換 NodeName |
| systemd 單元檔 | 必須確保 `ConditionPathExists=*` 不存在，否則 slurmd 無法啟動 |
| Include 指令 | 被包含的設定檔只有在檔名無路徑分隔符且位於 slurm.conf 旁邊時才會傳送 |
| Prolog/Epilog 腳本 | 只有在檔名無路徑分隔符且位於 slurm.conf 旁邊時才會傳送 |

---

### 注意事項

#### Slurm 25.11 新選項

```
SlurmctldParameters=reconfig_on_restart
```

此選項會讓每次 slurmctld 重新啟動（非由 `scontrol reconfigure` 觸發的程序重新啟動）都向所有 slurmd 和 sackd 守護程式觸發重新設定請求。對於無設定檔系統，這確保所有程序都執行目前的設定。

**警告**：如果您打算進行滾動升級，請謹慎使用，因為這可能導致守護程式比預期更早升級。

#### 設定來源優先順序

1. slurmd `--conf-server $host[:$port]` 選項
2. `-f $config_file` 選項
3. `SLURM_CONF` 環境變數（如果設定）
4. 本地 slurm 設定檔：
   - 預設 slurm 設定檔（通常 /etc/slurm.conf）
   - 使用者命令：快取的設定檔（run/slurm/conf/slurm.conf）
5. `SLURM_CONF_SERVER` 環境變數（如果設定）
6. DNS SRV 記錄（從最低優先順序值到最高）

**注意**：SRV 記錄的 TTL（存活時間）不影響取得的設定有效性。節點必須透過 `scontrol reconfig` 或 slurmd 重新啟動來通知設定變更。

#### 支援的設定檔

- slurm.conf
- acct_gather.conf
- cgroup.conf
- cli_filter.lua
- gres.conf
- helpers.conf
- job_container.conf
- mpi.conf
- namespace.yaml
- oci.conf
- plugstack.conf
- scrun.lua
- topology.conf
- topology.yaml

---

## 說明

### 無設定檔模式的優點

| 優點 | 說明 |
|------|------|
| 簡化部署 | 無需在每個節點預先部署設定檔 |
| 集中管理 | 所有設定集中在控制器上管理 |
| 一致性 | 確保所有節點使用相同設定 |
| 動態更新 | 透過 scontrol reconfig 即可更新所有節點 |

### 架構概念

```
                    ┌─────────────────┐
                    │   slurmctld     │
                    │  (設定來源)      │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │   slurmd     │  │   slurmd     │  │    sackd     │
    │  (運算節點)   │  │  (運算節點)   │  │  (登入節點)   │
    └──────────────┘  └──────────────┘  └──────────────┘
```

### DNS SRV 記錄與 HA

```
_slurmctld._tcp 3600 IN SRV <priority> <weight> <port> <host>
```

- **priority** - 優先順序（越低越優先）
- **weight** - 同優先順序的權重
- **port** - 連接埠（通常 6817）
- **host** - 主機名稱

---

## 實務範例

### 基本設定

**控制器 slurm.conf：**
```
SlurmctldParameters=enable_configless
SlurmctldHost=slurmctl-primary
SlurmctldHost=slurmctl-backup
```

**運算節點 systemd 單元檔：**
```ini
# /etc/systemd/system/slurmd.service
[Unit]
Description=Slurm node daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/slurmd --conf-server slurmctl-primary:6817,slurmctl-backup:6817 -D
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

### DNS SRV 記錄設定

**BIND 區域檔範例：**
```
; Slurm 控制器 SRV 記錄
_slurmctld._tcp.cluster.local. 3600 IN SRV 0 0 6817 slurmctl-primary.cluster.local.
_slurmctld._tcp.cluster.local. 3600 IN SRV 10 0 6817 slurmctl-backup.cluster.local.
```

### 使用 sackd 的登入節點

```bash
# 啟動 sackd 以減少設定檔請求
sackd --conf-server slurmctl-primary:6817
```

### 驗證設定

```bash
# 檢查 slurmd 是否正確取得設定
ls -la /run/slurm/conf/

# 檢查設定內容
cat /run/slurm/conf/slurm.conf | head -20

# 測試重新載入
scontrol reconfig
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未啟用 enable_configless | 在 slurm.conf 中設定 `SlurmctldParameters=enable_configless` |
| systemd 有 ConditionPathExists | 從單元檔中移除此行 |
| Include 檔案有路徑 | 將 Include 檔案放在 slurm.conf 旁邊，僅使用檔名 |
| Prolog/Epilog 有完整路徑 | 將腳本放在 slurm.conf 旁邊，僅使用檔名 |
| DNS SRV 優先順序錯誤 | 主要控制器應有最低優先順序值 |

### 疑難排解

```bash
# 檢查 DNS SRV 記錄
dig _slurmctld._tcp.cluster.local SRV

# 測試連接到控制器
nc -vz slurmctl-primary 6817

# 檢查 slurmd 日誌
journalctl -u slurmd -f

# 手動測試無設定檔啟動
slurmd --conf-server slurmctl-primary:6817 -D -vvv
```

### 效能考量

1. **登入節點流量**：
   - 大量使用者命令會產生設定檔請求
   - 使用 sackd 減少網路請求

2. **網路可靠性**：
   - 確保控制器網路穩定
   - 設定備用控制器

3. **DNS 快取**：
   - SRV 記錄 TTL 不影響設定有效性
   - 設定變更仍需 scontrol reconfig

---

## 快速參考

### slurm.conf 設定

```
# 啟用無設定檔模式
SlurmctldParameters=enable_configless

# 可選：重新啟動時重新設定（25.11+）
SlurmctldParameters=enable_configless,reconfig_on_restart
```

### slurmd 啟動選項

```bash
# 單一控制器
slurmd --conf-server slurmctl:6817

# 多控制器（HA）
slurmd --conf-server slurmctl-primary:6817,slurmctl-backup:6817

# 除錯模式
slurmd --conf-server slurmctl:6817 -D -vvv
```

### DNS SRV 記錄格式

```
_slurmctld._tcp.<domain> <TTL> IN SRV <priority> <weight> <port> <host>
```

### 設定檔位置

| 位置 | 說明 |
|------|------|
| `$SlurmdSpoolDir/conf-cache/` | 快取的設定檔 |
| `/run/slurm/conf/` | 設定檔符號連結 |

### 常用命令

| 命令 | 功能 |
|------|------|
| `scontrol reconfig` | 重新載入設定到所有節點 |
| `scontrol show config` | 顯示目前設定 |
| `dig _slurmctld._tcp SRV` | 檢查 DNS SRV 記錄 |

### 相關文件

- [sackd](sackd.html) - Slurm 認證與設定快取守護程式
- [快速入門管理指南](quickstart_admin.md) - 基本設定
- [高可用性](ha.html) - HA 設定
