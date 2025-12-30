# Slurm 帳務與資源限制

---

## TL;DR

Slurm 帳務系統透過 SlurmDBD（Slurm 資料庫常駐程式）記錄作業執行資訊。使用 `sacctmgr` 管理叢集、帳戶、使用者和關聯；`sacct` 查詢作業記錄；`sreport` 產生使用報表。需要 MySQL/MariaDB 資料庫支援。透過 `AccountingStorageEnforce` 參數可啟用限制強制執行（associations、limits、qos 等）。

---

## Translation（翻譯）

### 目錄

- [概述](#概述)
- [基礎設施](#基礎設施)
- [Slurm JobComp 配置](#slurm-jobcomp-配置)
- [建置前的 Slurm 帳務配置](#建置前的-slurm-帳務配置)
- [建置後的 Slurm 帳務配置](#建置後的-slurm-帳務配置)
- [SlurmDBD 配置](#slurmdbd-配置)
- [MySQL 配置](#mysql-配置)
- [SlurmDBD 歸檔與清除](#slurmdbd-歸檔與清除)
- [工具](#工具)
- [資料庫配置](#資料庫配置)
- [叢集選項](#叢集選項)
- [帳戶選項](#帳戶選項)
- [使用者選項](#使用者選項)
- [限制強制執行](#限制強制執行)
- [修改實體](#修改實體)
- [移除實體](#移除實體)
- [帳務資料解釋](#帳務資料解釋)

### 概述

Slurm 可以配置為收集每個作業和作業步驟執行的帳務資訊（accounting information）。帳務記錄可以寫入簡單的文字檔或資料庫。可以取得目前正在執行的作業和已終止作業的資訊。

**主要工具：**
- **sacct**：可以報告執行中或已終止作業的資源使用情況，包括個別任務，這對於檢測任務之間的負載不平衡非常有用
- **sstat**：僅用於查詢目前正在執行的作業狀態
- **sreport**：用於根據特定時間間隔內執行的所有作業產生報表

**與資源帳務相關的三種不同外掛類型：**

| 參數 | 功能 |
|------|------|
| **AccountingStorageType** | 控制如何記錄詳細的作業和作業步驟資訊。可以將此資訊儲存在文字檔或 SlurmDBD 中 |
| **JobAcctGatherType** | 與作業系統相關，控制使用什麼機制來收集帳務資訊。支援的值包括 `jobacct_gather/linux`、`jobacct_gather/cgroup` 和 `jobacct_gather/none` |
| **JobCompType** | 控制如何記錄作業完成資訊。可用於記錄基本作業資訊，如作業名稱、使用者名稱、分配的節點、開始時間、完成時間、退出狀態等 |

透過中間常駐程式發送資料可以提供更好的安全性和效能（透過快取資料）。SlurmDBD（Slurm 資料庫常駐程式）提供這樣的服務。SlurmDBD 使用 C 語言編寫，支援多執行緒，安全且快速。

**注意**：如果 SlurmDBD 已配置但未回應，則 slurmctld 將使用內部快取，直到 SlurmDBD 恢復服務。快取資料在關機時由 slurmctld 寫入本地儲存，並在啟動時恢復。

### 基礎設施

使用 SlurmDBD，我們能夠在單一位置收集來自多個叢集的資料。這對使用者命名和 ID 有一些限制：

- 帳務按使用者名稱（非使用者 ID）維護，但給定的使用者名稱應在所有電腦上指向同一個人
- 認證依賴使用者 ID 號碼，因此這些必須在與每個 SlurmDBD 通訊的所有電腦上統一
- 配置的 `SlurmUser` 必須在所有叢集上具有相同的名稱和 ID

**注意**：預設情況下只支援小寫使用者名稱，但您可以在 slurmdbd.conf 中配置 `Parameters=PreserveCaseUser` 以允許包含大寫字元的使用者名稱。

建議使用 [MUNGE](https://dun.github.io/munge/) 進行 SlurmDBD 通訊認證。

#### 儲存備援主機

可以透過在 slurm.conf 中指定 `AccountingStorageBackupHost` 以及在 slurmdbd.conf 中指定 `DbdBackupHost` 來配置 slurmdbd 的備援實例。備援主機應該在與主要 slurmdbd 實例不同的機器上。

### Slurm JobComp 配置

作業完成目前不支援 SlurmDBD，但可以直接寫入資料庫、腳本或平面檔案。重要參數包括：

| 參數 | 說明 |
|------|------|
| `JobCompHost` | 僅在使用資料庫時需要。資料庫伺服器執行的主機名稱或地址 |
| `JobCompLoc` | 僅在使用平面檔案時需要。寫入作業完成資料的檔案位置 |
| `JobCompPass` | 僅在使用資料庫時需要。連接資料庫的使用者密碼 |
| `JobCompPort` | 僅在使用資料庫時需要。資料庫接受通訊的網路埠 |
| `JobCompType` | 設定為 "jobcomp/mysql" 或 "jobcomp/filetxt" |
| `JobCompUser` | 僅在使用資料庫時需要。連接資料庫的使用者名稱 |

### 建置前的 Slurm 帳務配置

您可以透過使用 `AccountingStorageType=accounting_storage/slurmdbd` 來配置 SlurmDBD 與資料庫通訊。這允許建立稱為「關聯（associations）」的使用者實體，由叢集、使用者、帳戶和可選的分割區組成。

**MySQL 或 MariaDB 是首選的資料庫。**

**重要的 MySQL/MariaDB 伺服器設定：**

| 設定 | 建議值 | 說明 |
|------|--------|------|
| `innodb_buffer_pool_size` | 5-50% 可用記憶體，至少 4 GiB | 設定太小可能在升級大型 Slurm 資料庫或清除舊記錄時造成問題 |
| `innodb_log_file_size` | `innodb_buffer_pool_size` 的 25% / `innodb_log_files_in_group` | 減少不必要的小型磁碟寫入 |
| `innodb_lock_wait_timeout` | 900 秒 | 允許一些可能延長的查詢成功完成 |
| `max_allowed_packet` | 至少 16M | 使用舊版 SQL 伺服器時很重要 |

**my.cnf 範例：**
```ini
[mysqld]
innodb_buffer_pool_size=4096M
innodb_log_file_size=1024M
innodb_lock_wait_timeout=900
max_allowed_packet=16M
```

### 建置後的 Slurm 帳務配置

必須設定幾個 Slurm 配置參數以支援在 SlurmDBD 中歸檔資訊：

| 參數 | 說明 |
|------|------|
| `AccountingStorageEnforce` | 包含以逗號分隔的要強制執行的選項列表 |
| `AccountingStorageExternalHost` | 外部 slurmdbd 的逗號分隔列表 |
| `AccountingStorageHost` | SlurmDBD 執行的主機名稱或地址 |
| `AccountingStoragePass` | 用於存取資料庫的密碼 |
| `AccountingStoragePort` | SlurmDBD 接受通訊的網路埠 |
| `AccountingStorageType` | 設定為 "accounting_storage/slurmdbd" |
| `ClusterName` | 為每個 Slurm 管理的叢集設定唯一名稱 |
| `TrackWCKey` | 布林值。是否追蹤使用者的 wckey |

**AccountingStorageEnforce 選項：**

| 選項 | 功能 |
|------|------|
| `associations` | 如果使用者的關聯不在資料庫中，則阻止使用者執行作業 |
| `limits` | 強制執行關聯和 QOS 上設定的限制 |
| `nojobs` | 不在帳務中儲存作業資訊 |
| `nosteps` | 不在帳務中儲存步驟資訊 |
| `qos` | 要求所有作業指定有效的 QOS |
| `safe` | 確保只有在作業能夠執行完成時才啟動具有 TRES 分鐘限制的作業 |
| `wckeys` | 阻止使用者在沒有存取權的 wckey 下執行作業 |

### SlurmDBD 配置

SlurmDBD 需要自己的配置檔 "slurmdbd.conf"。此檔案應該只在 SlurmDBD 執行的電腦上，且只能由執行 SlurmDBD 的使用者讀取。

**重要參數：**

| 參數 | 說明 |
|------|------|
| `AuthInfo` | 如果使用第二個 MUNGE 常駐程式，儲存命名 socket 的路徑名稱 |
| `AuthType` | 定義認證方法，建議 "auth/munge" |
| `DbdHost` | Slurm 資料庫常駐程式執行的機器名稱 |
| `DbdPort` | SlurmDBD 監聽的埠號（預設 6819） |
| `LogFile` | 日誌檔案的完整路徑名稱 |
| `PluginDir` | 尋找 Slurm 外掛的位置 |
| `SlurmUser` | slurmdbd 常駐程式執行的使用者名稱 |
| `StorageHost` | 資料庫執行的主機名稱 |
| `StorageLoc` | 寫入帳務記錄的資料庫名稱（預設 slurm_acct_db） |
| `StoragePass` | 存取資料庫的密碼 |
| `StoragePort` | 資料庫監聽的埠 |
| `StorageType` | 必須設定為 "accounting_storage/mysql" |
| `StorageUser` | 連接資料庫的使用者名稱 |

### MySQL 配置

雖然 Slurm 會自動建立資料庫表，但您需要確保 StorageUser 在 MySQL 或 MariaDB 資料庫中被授予權限。

```sql
-- 建立使用者
mysql> create user 'slurm'@'localhost' identified by 'password';

-- 授予權限
mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost';

-- 驗證 InnoDB 支援
mysql> SHOW ENGINES;

-- 建立資料庫
mysql> create database slurm_acct_db;
```

### SlurmDBD 歸檔與清除

隨著時間推移，slurm 資料庫可能會增長到難以管理的大小。為了將資料庫維持在合理的大小，slurmdbd 支援根據資料年齡進行歸檔和清除。

**強烈建議在設定帳務後不久就制定資料保留計劃。**

歸檔和清除選項的形式為 `Archive${*}` 和 `Purge${*}After`。

單位很重要。例如：
- `PurgeJobsAfter=12months` 將在每月初清除超過 12 個月的作業
- `PurgeJobsAfter=365days` 將在每天初清除超過 365 天的作業

#### 歸檔伺服器

如果您的站點需要持續存取已歸檔/清除的資料，可以建立 slurmdbd 的歸檔實例。歸檔實例的 slurmdbd 不應與生產伺服器通訊。

### 工具

Slurm 包含幾個讓您處理帳務資料的工具：

| 工具 | 功能 |
|------|------|
| **sacct** | 用於檢索儲存在資料庫中的執行中和已完成作業的詳細資訊 |
| **sacctmgr** | 用於管理資料庫中的實體，包括叢集、帳戶、使用者關聯、QOS 等 |
| **sreport** | 用於產生在給定時間段內收集的使用情況的各種報表 |

**第三方視覺化工具：**
- **Grafana**：允許使用 Prometheus 或 InfluxDB 收集的資料建立帶有各種圖表的儀表板
- **InfluxDB**：包含從 Slurm 收集效能指標的匯出工具
- **Prometheus**：包含從 Slurm 收集效能指標的匯出工具

### 資料庫配置

帳務記錄基於我們稱為**關聯（Association）**的內容來維護，由四個元素組成：叢集、帳戶、使用者名稱和可選的分割區名稱。使用 `sacctmgr` 指令來建立和管理這些記錄。

**注意**：設定帳務關聯有順序。您必須先定義叢集，然後才能新增帳戶，必須先新增帳戶，然後才能新增使用者。

```bash
# 新增叢集
sacctmgr add cluster snowflake

# 新增帳戶
sacctmgr add account none,test Cluster=snowflake \
  Description="none" Organization="none"

# 新增階層式帳戶
sacctmgr add account science \
 Description="science accounts" Organization=science
sacctmgr add account chemistry,physics parent=science \
 Description="physical sciences" Organization=science

# 新增使用者
sacctmgr add user brian Account=physics
sacctmgr add user da DefaultAccount=test
```

### 叢集選項

| 選項 | 說明 |
|------|------|
| `Name=` | 叢集名稱 |

### 帳戶選項

| 選項 | 說明 |
|------|------|
| `Cluster=` | 只將此帳戶新增到這些叢集 |
| `Description=` | 帳戶描述（預設為帳戶名稱） |
| `Name=` | 帳戶名稱（必須唯一） |
| `Organization=` | 帳戶的組織 |
| `Parent=` | 使此帳戶成為其他帳戶的子帳戶 |

### 使用者選項

| 選項 | 說明 |
|------|------|
| `Account=` | 要新增使用者的帳戶 |
| `AdminLevel=` | 管理權限等級（None、Operator、Admin） |
| `Cluster=` | 只新增到這些叢集的帳戶 |
| `DefaultAccount=` | 使用者的預設帳戶（建立時必需） |
| `DefaultWCKey=` | 使用者的預設 wckey |
| `Name=` | 使用者名稱 |
| `NewName=` | 用於在帳務資料庫中重新命名使用者 |
| `Partition=` | 此關聯適用的 Slurm 分割區名稱 |

### 限制強制執行

要啟用任何限制強制執行，您必須在 slurm.conf 中至少有 `AccountingStorageEnforce=limits`。否則，即使您設定了限制，它們也不會被強制執行。

### 修改實體

修改實體時，您可以以類似 SQL 的方式指定許多不同的選項：

```bash
sacctmgr modify <entity> set <options> where <options>

# 範例：將預設帳戶從 test 改為 none
sacctmgr modify user set default=none where default=test
```

### 移除實體

```bash
# 移除預設帳戶為 test 的所有使用者
sacctmgr remove user where default=test

# 從 physics 帳戶移除使用者 brian
sacctmgr remove user brian where account=physics
```

**注意**：在大多數情況下，移除的實體會保留在 slurm 資料庫中，但標記為已刪除。如果實體存在不到 1 天，該實體將被完全移除。

### 帳務資料解釋

Slurm 帳務主要專注於平行運算。因此，它在「任務層級」收集統計資料。任務是在步驟中執行的一組使用者程序，而步驟是作業的一部分。

JobAcctGather 外掛在 slurm.conf 中定義的 `JobAcctGatherFrequency` 間隔收集某些 TRES 的指標。

對於單一任務程序，`TresUsageInTot` 可用作步驟在任何時間消耗的最大記憶體，這將等於 `MaxRSS` 並有效地表示步驟記憶體峰值。

---

## Explanation（解釋）

### 帳務系統架構

```
┌─────────────────────────────────────────────────────────────────┐
│                    Slurm 帳務系統架構                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  使用者/管理員                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │  sacct   │  │sacctmgr  │  │ sreport  │                      │
│  │(查詢作業)│  │(管理實體)│  │(產生報表)│                      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                      │
│       │             │             │                             │
│       └─────────────┼─────────────┘                             │
│                     │                                           │
│                     ▼                                           │
│  ┌─────────────────────────────────────┐                       │
│  │           SlurmDBD                   │                       │
│  │     (Slurm 資料庫常駐程式)           │                       │
│  │                                      │                       │
│  │  • 集中管理帳務資料                  │                       │
│  │  • 支援多叢集                        │                       │
│  │  • 提供快取和安全性                  │                       │
│  └──────────────┬──────────────────────┘                       │
│                 │                                               │
│                 ▼                                               │
│  ┌─────────────────────────────────────┐                       │
│  │      MySQL / MariaDB 資料庫          │                       │
│  │                                      │                       │
│  │  表格：                              │                       │
│  │  • *_assoc_table (關聯)             │                       │
│  │  • *_job_table (作業)               │                       │
│  │  • *_step_table (步驟)              │                       │
│  │  • qos_table (QOS)                  │                       │
│  │  • tres_table (TRES)                │                       │
│  └─────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### 關聯（Association）的階層結構

```
叢集 (Cluster)
    │
    └── 帳戶 (Account) - 可以階層式
            │
            ├── 子帳戶 (Child Account)
            │       │
            │       └── 使用者 (User)
            │               │
            │               └── 分割區關聯 (Partition Association)
            │
            └── 使用者 (User)
                    │
                    └── 分割區關聯 (Partition Association)

範例：
snowflake (叢集)
    │
    ├── science (帳戶)
    │       │
    │       ├── chemistry (子帳戶)
    │       │       └── alice (使用者)
    │       │
    │       └── physics (子帳戶)
    │               └── bob (使用者)
    │
    └── engineering (帳戶)
            └── charlie (使用者)
```

---

## Practical Example（實用範例）

### 範例 1：初始設定帳務系統

```bash
# 1. 配置 slurmdbd.conf
cat > /etc/slurm/slurmdbd.conf << 'EOF'
AuthType=auth/munge
DbdHost=localhost
DbdPort=6819
SlurmUser=slurm
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageLoc=slurm_acct_db
StorageUser=slurm
StoragePass=your_password
EOF

# 2. 設定權限
chmod 600 /etc/slurm/slurmdbd.conf
chown slurm:slurm /etc/slurm/slurmdbd.conf

# 3. 啟動 slurmdbd
systemctl start slurmdbd

# 4. 新增叢集
sacctmgr add cluster mycluster
```

### 範例 2：建立帳戶階層

```bash
# 建立頂層帳戶
sacctmgr add account research \
    Description="Research Division" \
    Organization="MyOrg"

# 建立子帳戶
sacctmgr add account hpc parent=research \
    Description="HPC Team" \
    Organization="MyOrg"

sacctmgr add account ai parent=research \
    Description="AI Team" \
    Organization="MyOrg"

# 查看帳戶結構
sacctmgr show account format=Account,Description,Organization,ParentName
```

### 範例 3：新增使用者並設定限制

```bash
# 新增使用者到帳戶
sacctmgr add user alice DefaultAccount=hpc

# 設定使用者限制
sacctmgr modify user alice set \
    MaxJobs=10 \
    MaxSubmitJobs=50 \
    MaxWallDurationPerJob=72:00:00

# 查看使用者資訊
sacctmgr show user alice withassoc
```

### 範例 4：使用 sacct 查詢作業

```bash
# 查看最近的作業
sacct -u $USER --starttime=2024-01-01

# 查看特定作業的詳細資訊
sacct -j 12345 --format=JobID,JobName,Partition,Account,AllocCPUS,State,ExitCode,Elapsed

# 查看作業的資源使用
sacct -j 12345 --format=JobID,MaxRSS,MaxVMSize,AveRSS,AveCPU

# 查看所有欄位
sacct -j 12345 -o ALL
```

### 範例 5：使用 sreport 產生報表

```bash
# 叢集使用率報表
sreport cluster utilization start=2024-01-01 end=2024-01-31

# 使用者使用報表
sreport user top start=2024-01-01 end=2024-01-31 TopCount=20

# 帳戶使用報表
sreport account UserUtilizationByAccount start=2024-01-01 end=2024-01-31

# GPU 使用報表
sreport cluster AccountUtilizationByUser start=2024-01-01 tres=gres/gpu
```

### 範例 6：配置限制強制執行

```bash
# 在 slurm.conf 中啟用限制
# AccountingStorageEnforce=associations,limits,qos

# 設定 QOS
sacctmgr add qos normal priority=50 MaxWall=24:00:00
sacctmgr add qos high priority=100 MaxWall=72:00:00 MaxJobsPA=5

# 將 QOS 分配給帳戶
sacctmgr modify account hpc set qos=normal,high

# 設定預設 QOS
sacctmgr modify account hpc set DefaultQOS=normal
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| slurmdbd.conf 權限錯誤 | 包含密碼的配置檔被其他使用者讀取 | 設定 `chmod 600 slurmdbd.conf` |
| MySQL buffer pool 太小 | 升級或清除時效能問題 | 設定至少 4GB |
| 未先新增叢集 | 無法新增帳戶或使用者 | 先執行 `sacctmgr add cluster` |
| SlurmUser 不一致 | 認證失敗 | 確保所有叢集和 slurmdbd 使用相同的 SlurmUser |
| 未設定 AccountingStorageEnforce | 限制不生效 | 在 slurm.conf 中設定適當的值 |

### ✅ 實用技巧

1. **驗證帳務設定**
   ```bash
   # 檢查 slurmdbd 狀態
   sacctmgr show config

   # 檢查叢集註冊
   sacctmgr list cluster

   # 測試資料庫連線
   sacctmgr show stats
   ```

2. **定期備份資料庫**
   ```bash
   # 備份整個資料庫
   mysqldump -u slurm -p slurm_acct_db > slurm_backup_$(date +%Y%m%d).sql
   ```

3. **監控資料庫大小**
   ```bash
   # 查看表格大小
   mysql -u slurm -p -e "SELECT table_name,
       round(data_length/1024/1024,2) as 'Data (MB)'
       FROM information_schema.tables
       WHERE table_schema='slurm_acct_db';"
   ```

4. **設定資料清除策略**
   ```bash
   # 在 slurmdbd.conf 中
   PurgeJobAfter=12months
   PurgeStepAfter=6months
   PurgeEventAfter=12months
   ```

5. **除錯帳務問題**
   ```bash
   # 啟用除錯模式
   slurmdbd -D -vvv

   # 檢查日誌
   tail -f /var/log/slurm/slurmdbd.log
   ```

---

## Quick Reference（快速參考）

### sacctmgr 常用指令

| 指令 | 功能 |
|------|------|
| `sacctmgr add cluster <name>` | 新增叢集 |
| `sacctmgr add account <name>` | 新增帳戶 |
| `sacctmgr add user <name>` | 新增使用者 |
| `sacctmgr modify user <name> set ...` | 修改使用者 |
| `sacctmgr remove user <name>` | 移除使用者 |
| `sacctmgr show assoc` | 顯示關聯 |
| `sacctmgr show config` | 顯示配置 |

### sacct 常用選項

| 選項 | 功能 |
|------|------|
| `-j <jobid>` | 指定作業 ID |
| `-u <user>` | 指定使用者 |
| `-S <starttime>` | 開始時間 |
| `-E <endtime>` | 結束時間 |
| `--format=<fields>` | 輸出格式 |
| `-X` | 不顯示作業步驟 |

### sreport 常用報表

| 報表 | 指令 |
|------|------|
| 叢集使用率 | `sreport cluster utilization` |
| 使用者排名 | `sreport user top` |
| 帳戶使用 | `sreport account UserUtilizationByAccount` |
| 作業統計 | `sreport job SizesByAccount` |

### 配置檔位置

| 檔案 | 用途 |
|------|------|
| `/etc/slurm/slurm.conf` | Slurm 主配置 |
| `/etc/slurm/slurmdbd.conf` | SlurmDBD 配置 |
| `/var/log/slurm/slurmdbd.log` | SlurmDBD 日誌 |

### AccountingStorageEnforce 選項

| 選項 | 效果 |
|------|------|
| `associations` | 要求有效的關聯 |
| `limits` | 強制執行限制 |
| `qos` | 要求有效的 QOS |
| `safe` | 安全模式（確保作業能完成） |
| `wckeys` | 要求有效的 WCKey |
| `nojobs` | 不儲存作業資訊 |
| `nosteps` | 不儲存步驟資訊 |
