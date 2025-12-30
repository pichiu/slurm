# Slurm 管理員快速入門指南 (Quick Start Administrator Guide)

## TL;DR

Slurm 管理員快速入門：同步叢集時鐘和使用者、安裝 MUNGE 認證、下載並安裝 Slurm、使用設定工具產生 slurm.conf、啟動 slurmctld 和 slurmd 守護程式。Slurm 叢集包含計算節點、控制節點、資料庫節點和登入節點等不同類型。

---

## 翻譯

### 概觀

請參閱[使用者快速入門指南](../users/quickstart.md)以獲得一般概觀。

另請參閱[支援平台](platforms.html)以獲得支援的電腦平台清單。

關於執行升級的資訊，請參閱[升級指南](upgrades.html)。

---

### 超快速入門

1. 確保叢集中所有節點的時鐘、使用者和群組（UID 和 GID）已同步。
2. 安裝 [MUNGE](https://dun.github.io/munge/) 進行認證。確保叢集中所有節點都有相同的 `munge.key`。確保 MUNGE 守護程式 `munged` 在啟動 Slurm 守護程式之前已啟動。
3. [下載](https://www.schedmd.com/download-slurm/)最新版本的 Slurm。
4. 使用以下方法之一安裝 Slurm：
   - 建置 RPM 或 DEB 套件（建議用於生產環境）
   - 從原始碼手動建置（適用於開發人員或進階使用者）
   - **注意**：某些 Linux 發行版可能在軟體庫中提供**非官方** Slurm 套件。SchedMD 不維護也不推薦這些套件。
5. 使用網頁瀏覽器和 [Slurm 設定工具](configurator.html)建立設定檔。
   - **注意**：`SlurmUser` 必須在啟動 Slurm 之前存在，且必須存在於叢集的所有節點上。
   - **注意**：Slurm 的日誌檔、程序 ID 檔、狀態儲存目錄等的父目錄不會由 Slurm 建立。它們必須在啟動 Slurm 守護程式之前建立並設定為 `SlurmUser` 可寫入。
6. 將設定檔安裝到 `<sysconfdir>/slurm.conf`。
   - **注意**：您需要在叢集的所有節點上安裝此設定檔。
7. systemd（選用）：在每個系統上啟用適當的服務：
   - 控制器：`systemctl enable slurmctld`
   - 資料庫：`systemctl enable slurmdbd`
   - 計算節點：`systemctl enable slurmd`
8. 啟動 `slurmctld` 和 `slurmd` 守護程式。

---

### 建置與安裝 Slurm

#### 安裝先決條件

在建置 Slurm 之前，請考慮您的安裝需要哪些外掛程式。建置哪些外掛程式取決於執行 configure 腳本時可用的函式庫。

**注意**：在大多數情況下，所需的套件是對應的開發函式庫，其確切名稱可能因發行版而異。RHEL 系列發行版的典型命名慣例是 `NAME-devel`，而 Debian 系列發行版的慣例是 `libNAME-dev`。

| 元件 | 所需開發函式庫 |
|------|----------------|
| `acct_gather_energy/ipmi` - 透過 IPMI 收集能源消耗 | freeipmi |
| `acct_gather_interconnect/ofed` - 收集 InfiniBand 網路流量資料 | libibmad, libibumad |
| `acct_gather_profile/hdf5` - 透過 HDF5 收集詳細作業剖析 | hdf5 |
| `accounting_storage/mysql` - 計費功能必須 | MySQL 或 MariaDB |
| `auth/slurm` - 替代 MUNGE 的認證方法 | jwt |
| `auth/munge` - 預設認證方法 | MUNGE |
| `AutoDetect=nvml` - 自動偵測 NVIDIA GPU | libnvidia-ml |
| `AutoDetect=oneapi` - 自動偵測 Intel GPU | libvpl |
| `AutoDetect=rsmi` - 自動偵測 AMD GPU | ROCm |
| HTML man pages | man2html |
| Lua API | lua |
| PAM 支援 | PAM |
| PMIx 支援 | pmix |
| scontrol/sacctmgr 互動模式的 Readline | readline |
| slurmrestd | http_parser, json-c, yaml（選用）, jwt（選用）|
| sview | gtk+-2.0 |
| NUMA 支援 | numa |
| task/cgroup | hwloc, bpf（cgroup/v2）, dbus（cgroup/v2）|

#### 建置 RPM

要直接建置 RPM，請將發行的 tarball 複製到目錄中並執行：

```bash
rpmbuild -ta slurm-23.02.7.tar.bz2
```

rpm 檔案將安裝在建置它們的使用者的 `$(HOME)/rpmbuild` 目錄下。

您可以使用家目錄中的 `.rpmmacros` 檔案控制 RPM 建置的某些方面：

```
# .rpmmacros
%_enable_debug     "--with-debug"
%_prefix           /opt/slurm
%_slurm_sysconfdir %{_prefix}/etc/slurm
%_defaultdocdir    %{_prefix}/doc
%with_munge        "--with-munge=/opt/munge"
```

#### 建置 Debian 套件

從 Slurm 23.11.0 開始，Slurm 包含建置 Debian 套件所需的檔案：

```bash
# 安裝基本 Debian 套件建置需求
apt-get install build-essential fakeroot devscripts equivs

# 解壓縮發行的 tarball
tar -xaf slurm*tar.bz2

# 切換到包含 Slurm 原始碼的目錄
cd slurm-*

# 安裝 Slurm 套件相依性
mk-build-deps -i debian/control

# 建置 Slurm 套件
debuild -b -uc -us
```

#### 安裝套件

以下套件建議用於不同節點類型的基本功能：

| RPM 名稱 | DEB 名稱 | 登入 | 控制器 | 計算 | DBD |
|----------|----------|------|--------|------|-----|
| slurm | slurm-smd | ✓ | ✓ | ✓ | ✓ |
| slurm-perlapi | slurm-smd-client | ✓ | ✓ | ✓ | |
| slurm-slurmctld | slurm-smd-slurmctld | | ✓ | | |
| slurm-slurmd | slurm-smd-slurmd | | | ✓ | |
| slurm-slurmdbd | slurm-smd-slurmdbd | | | | ✓ |

**處理相依性**：建議避免使用低階命令如 `dpkg` 和 `rpm`，改用高階命令如 `dnf` 和 `apt` 進行所有操作。

#### 手動建置

```bash
# 解壓縮發行的 tarball
tar -xaf slurm*tar.bz2

# 切換到目錄並設定
cd slurm-*
./configure --prefix=/usr/local --sysconfdir=/etc/slurm

# 編譯和安裝
make install

# 更新函式庫快取
ldconfig -n /usr/local/lib64
```

常用的 configure 選項：
- `--enable-debug`：啟用 Slurm 內的額外除錯邏輯
- `--prefix=PREFIX`：安裝架構獨立檔案的路徑（預設 /usr/local）
- `--sysconfdir=DIR`：指定 Slurm 設定檔位置

---

### 節點類型

叢集由多種不同類型的節點組成。至少需要一個計算節點和一個控制節點才能運作。

大多數 Slurm 守護程式應以非 root 服務帳戶執行。我們建議您建立一個名為 `slurm` 的 Unix 使用者供 slurmctld 使用，並確保它存在於整個叢集中。

#### 計算節點 (Compute Node)

計算節點執行叢集中的計算工作。`slurmd` 守護程式在每個計算節點上執行。它監控節點上所有執行的任務、接受工作、啟動任務並根據請求終止執行中的任務。因為 slurmd 啟動和管理使用者作業，它必須以 root 使用者執行。

#### 控制節點 (Controller Node)

執行 `slurmctld` 的機器有時稱為「頭節點」或「控制器」。它協調 Slurm 活動，包括作業佇列管理、監控節點狀態和分配資源給作業。有一個選用的備份控制器，在主控制器失敗時自動接管（見高可用性章節）。

#### DBD 節點 (Database Node)

如果您想將作業計費記錄儲存到資料庫，應使用 `slurmdbd`（Slurm 資料庫守護程式）。建議在與控制器不同的機器上執行 slurmdbd 守護程式。在較大的系統上，我們也建議 slurmdbd 使用的資料庫在單獨的機器上。

#### 登入節點 (Login Node)

登入節點或提交主機是用於存取叢集的共用系統。使用者可以使用登入節點來準備資料、準備作業提交、提交作業、檢查工作狀態等。登入節點應能存取使用者預期使用的任何 Slurm 用戶端命令。

#### REST API 節點 (Restd Node)

`slurmrestd` 守護程式可部署以提供 REST API，用於以程式方式與 Slurm 叢集互動。

---

### 高可用性

可以設定多個 SlurmctldHost 項目，第一個以外的任何項目都被視為備份主機。所有主機應掛載包含狀態資訊的共用檔案系統。

如果指定多個主機，當主要節點失敗時，第二個列出的 SlurmctldHost 將接管。當主要節點恢復服務時，它會通知備份節點。備份節點然後儲存狀態並返回備份模式。

slurmdbd 的備份實例也可以透過在 slurm.conf 中指定 `AccountingStorageBackupHost` 以及在 slurmdbd.conf 中指定 `DbdBackupHost` 來設定。

---

### 基礎設施

#### 使用者和群組識別

整個叢集必須有統一的使用者和群組名稱空間（包括 UID 和 GID）。不需要允許使用者登入控制主機，但使用者和群組必須在這些主機上可解析。

#### Slurm 通訊認證

Slurm 元件之間的所有通訊都經過認證。認證基礎設施由動態載入的外掛程式提供，透過 Slurm 設定檔中的 `AuthType` 關鍵字在執行時選擇。

- **auth/munge**（預設）：使用 MUNGE 進行認證，需要安裝 MUNGE 套件
- **auth/slurm**（23.11.0+）：內部認證外掛程式，需要 jwt 套件

MUNGE 目前是預設和推薦的選項。

#### MPI 支援

Slurm 支援許多不同的 MPI 實作。更多資訊請參閱 [MPI 指南](../users/mpi_guide.md)。

#### 排程器支援

Slurm 可以根據您的需求設定相當簡單或相當複雜的排程演算法。

**PriorityType** 選項：
- `basic`：先進先出
- `multifactor`：根據多種設定參數（年齡、大小、公平共享分配等）分配作業優先順序

**SchedType** 選項：
- `builtin`：嚴格按優先順序啟動作業
- `backfill`：如果不會延遲較高優先順序作業的預期啟動時間，則啟動較低優先順序的作業
- `gang`：在同一分割區/佇列中對作業進行時間片段

#### 資源選擇

Slurm 使用的資源選擇機制由 `SelectType` 設定參數控制。如果您想在每個節點上執行多個作業，但追蹤和管理處理器、記憶體和其他資源的分配，建議使用 `cons_tres`（可消耗可追蹤資源）外掛程式。

---

### 設定

Slurm 設定檔包含各種參數。此設定檔必須在叢集的每個節點上可用且內容一致。

範例 slurm.conf：

```
#
# Sample /etc/slurm.conf
#
SlurmctldHost=mcri(12.34.56.78)
SlurmctldHost=mcrj(12.34.56.79)
#
AuthType=auth/munge
Epilog=/usr/local/slurm/etc/epilog
PluginDir=/usr/local/slurm/lib/slurm
Prolog=/usr/local/slurm/etc/prolog
SchedulerType=sched/backfill
SelectType=select/linear
SlurmUser=slurm
SlurmctldPort=7002
SlurmctldTimeout=300
SlurmdPort=7003
SlurmdSpoolDir=/var/spool/slurmd.spool
SlurmdTimeout=300
StateSaveLocation=/var/spool/slurm.state
TreeWidth=16
#
# Node Configurations
#
NodeName=DEFAULT CPUs=2 RealMemory=2000 TmpDisk=64000 State=UNKNOWN
NodeName=mcr[0-1151] NodeAddr=emcr[0-1151]
#
# Partition Configurations
#
PartitionName=DEFAULT State=UP
PartitionName=pdebug Nodes=mcr[0-191] MaxTime=30 MaxNodes=32 Default=YES
PartitionName=pbatch Nodes=mcr[192-1151]
```

**StateSaveLocation**：用於儲存叢集當前狀態的資訊。此目錄應位於低延遲本地磁碟上，以防止檔案系統延遲影響 Slurm 效能。

---

### 安全性

除了基於 `AuthType` 值的 Slurm 通訊認證外，數位簽章也用於作業步驟憑證。數位簽章機制由 `CredType` 設定參數指定，預設機制是 MUNGE。

**PAM 支援**：可使用 PAM 模組（可插拔認證模組）來防止使用者存取未分配給該使用者的節點。

---

### 啟動守護程式

用於測試目的，您可能想先在一個節點上只執行 slurmctld 和 slurmd。預設情況下，它們在背景執行。使用 `-D` 選項讓守護程式在前景執行並將日誌輸出到終端機。`-v` 選項會記錄更詳細的事件。

```bash
# 視窗 1：啟動控制器
slurmctld -D -vvvvvv

# 視窗 2：啟動計算節點守護程式
slurmd -D -vvvvv

# 視窗 3：測試功能
srun -N1 /bin/hostname
```

另一個重要選項是 `-c` 用於清除先前的狀態資訊。沒有 `-c` 選項時，守護程式會還原任何先前儲存的狀態資訊。

---

### 管理範例

`scontrol` 可用於列印所有系統資訊和修改大部分資訊。

```bash
# 列印所有作業的詳細狀態
scontrol show job

# 列印特定作業的狀態並變更優先順序
scontrol show job 477
scontrol update JobId=477 Priority=0

# 列印節點狀態並將節點設為維護模式
scontrol show node adev13
scontrol update NodeName=adev13 State=DRAIN

# 將節點恢復服務
scontrol update NodeName=adev13 State=IDLE

# 重新設定所有 Slurm 守護程式
scontrol reconfig

# 列印目前設定
scontrol show config

# 關閉所有 Slurm 守護程式
scontrol shutdown
```

---

## 說明

### 節點類型比較

| 節點類型 | 守護程式 | 執行身份 | 主要功能 |
|----------|----------|----------|----------|
| 計算節點 | slurmd | root | 執行作業、監控任務 |
| 控制節點 | slurmctld | slurm | 作業排程、資源分配 |
| DBD 節點 | slurmdbd | slurm | 計費資料庫管理 |
| 登入節點 | 無/sackd | - | 作業提交、狀態查詢 |
| REST 節點 | slurmrestd | slurm | REST API 服務 |

### 設定檔關係

```
slurm.conf        - 主要設定檔（所有節點）
slurmdbd.conf     - 資料庫守護程式設定（DBD 節點）
gres.conf         - 通用資源設定（計算節點）
cgroup.conf       - Cgroups 設定（計算節點）
```

---

## 實務範例

### 最小叢集設定步驟

```bash
# 1. 安裝 MUNGE
yum install munge munge-libs munge-devel  # RHEL
apt install munge libmunge-dev            # Debian

# 2. 產生並分發 MUNGE 金鑰
create-munge-key
scp /etc/munge/munge.key node1:/etc/munge/
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key

# 3. 啟動 MUNGE
systemctl enable --now munge

# 4. 建立 Slurm 使用者
useradd -r -s /sbin/nologin slurm

# 5. 建立必要目錄
mkdir -p /var/spool/slurm/ctld /var/spool/slurm/d
mkdir -p /var/log/slurm
chown -R slurm:slurm /var/spool/slurm /var/log/slurm

# 6. 安裝 Slurm 套件
yum install slurm slurm-slurmctld slurm-slurmd  # RHEL

# 7. 建立設定檔（使用設定工具或手動）
# 複製到 /etc/slurm/slurm.conf

# 8. 啟動服務
systemctl enable --now slurmctld  # 控制節點
systemctl enable --now slurmd     # 計算節點
```

### 驗證安裝

```bash
# 檢查守護程式狀態
systemctl status slurmctld
systemctl status slurmd

# 檢查節點狀態
sinfo

# 檢查控制器狀態
scontrol ping

# 測試作業提交
srun -N1 hostname
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 忘記同步 UID/GID | 確保所有節點上的使用者和群組 ID 一致 |
| MUNGE 金鑰不一致 | 使用相同的 munge.key 檔案分發到所有節點 |
| 未建立必要目錄 | 在啟動守護程式前建立日誌和狀態目錄 |
| StateSaveLocation 使用 NFS | 使用低延遲本地磁碟或專用共用儲存 |
| slurmd 未以 root 執行 | slurmd 必須以 root 執行 |
| slurmctld 以 root 執行 | slurmctld 應以 SlurmUser（非 root）執行 |

### 安裝前檢查清單

- [ ] 所有節點時鐘已同步（NTP）
- [ ] UID/GID 在所有節點上一致
- [ ] SlurmUser 在所有節點上存在
- [ ] MUNGE 已安裝並執行
- [ ] munge.key 在所有節點上相同
- [ ] 必要目錄已建立且權限正確
- [ ] 防火牆允許 Slurm 通訊埠

### 除錯技巧

```bash
# 以前景模式啟動守護程式並查看日誌
slurmctld -D -vvvv
slurmd -D -vvvv

# 檢查設定語法
slurmd -C  # 檢查計算節點設定

# 測試 MUNGE 認證
munge -n | unmunge

# 檢查網路連線
scontrol ping
```

---

## 快速參考

### 重要守護程式

| 守護程式 | 功能 | 設定檔 |
|----------|------|--------|
| slurmctld | 控制器守護程式 | slurm.conf |
| slurmd | 計算節點守護程式 | slurm.conf |
| slurmdbd | 資料庫守護程式 | slurmdbd.conf |
| slurmrestd | REST API 守護程式 | slurm.conf |

### 重要設定參數

| 參數 | 說明 | 範例 |
|------|------|------|
| SlurmctldHost | 控制節點主機名 | SlurmctldHost=head01 |
| SlurmUser | Slurm 服務帳戶 | SlurmUser=slurm |
| AuthType | 認證類型 | AuthType=auth/munge |
| StateSaveLocation | 狀態儲存目錄 | StateSaveLocation=/var/spool/slurm |
| SlurmdSpoolDir | slurmd 暫存目錄 | SlurmdSpoolDir=/var/spool/slurmd |

### systemctl 命令

```bash
# 啟用並啟動服務
systemctl enable --now slurmctld
systemctl enable --now slurmd
systemctl enable --now slurmdbd

# 檢查狀態
systemctl status slurmctld

# 重新啟動
systemctl restart slurmctld

# 查看日誌
journalctl -u slurmctld -f
```

### 常用管理命令

| 命令 | 功能 |
|------|------|
| `scontrol show config` | 顯示目前設定 |
| `scontrol reconfig` | 重新載入設定 |
| `scontrol ping` | 測試控制器連線 |
| `scontrol shutdown` | 關閉守護程式 |
| `sinfo` | 顯示節點和分割區狀態 |
| `squeue` | 顯示作業佇列 |
