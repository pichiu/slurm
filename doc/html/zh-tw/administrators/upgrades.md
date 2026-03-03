# Slurm 升級指南 (Upgrade Guide)

## TL;DR

Slurm 支援特定版本間的原地升級。升級順序：slurmdbd → slurmctld → slurmd → 其他命令。24.11 版本起支援與前三個主要版本的相容性。升級前務必備份資料庫和 StateSaveLocation。降級不受支援，一旦啟動新版守護程式後，狀態檔案和資料庫會更新為新格式。

---

## 翻譯

### 發行週期

Slurm 版本號包含三個以句點分隔的數字，代表主要版本和維護版本。例如 Slurm 23.11.4：

- **23.11** = 主要版本
  - 對應初次發行的年份和月份（2023 年 11 月）
  - 主要版本可能包含 RPC、狀態檔案、設定選項和核心功能的變更
- **.4** = 維護版本
  - 維護版本可能包含錯誤修正和效能改進

從 24.05 版本開始，Slurm 從 9 個月的發行週期改為 6 個月的發行週期。

#### 相容性視窗

Slurm 長久以來支援從前兩個主要版本的原地升級。隨著 2024 年改為 6 個月發行週期，Slurm 24.11 引入與**前三個主要版本**的相容性。例如，slurmdbd 25.11.x 能夠接受來自版本 25.11.x、25.05.x、24.11.x 或 24.05.x 的 slurmctld 守護程式和命令的訊息。

| Slurm 版本 | 支援結束時間 | 相容的先前版本 |
|------------|--------------|----------------|
| 23.11 | 2025 年 5 月（18 個月）| 23.02, 22.05 |
| 24.05 | 2025 年 11 月（18 個月）| 23.11, 23.02 |
| 24.11 | 2026 年 5 月（18 個月）| 24.05, 23.11, 23.02 |
| 25.05 | 2026 年 11 月（18 個月）| 24.11, 24.05, 23.11 |
| 25.11 | 2027 年 5 月（18 個月）| 25.05, 24.11, 24.05 |

從不相容版本的升級會在啟動時立即失敗。必須分步驟執行升級，先升級到與目前執行版本相容的較新版本。

#### EPEL 儲存庫

2021 年初，Slurm 被加入 EPEL 儲存庫。此版本不由 SchedMD 提供或支援。為防止 Slurm 被意外變更，建議在 `/etc/yum.repos.d/epel.repo` 的 `[epel]` 區段下新增：

```
exclude=slurm*
```

#### 預發行版本

安裝預發行版本（如 24.05.0rc1）時，應準備好面對意外當機、錯誤和狀態資訊遺失。這些版本僅經過有限測試，不適用於生產叢集。

---

### 還原升級

一旦任何 Slurm 守護程式啟動後，**不支援**還原升級（或降級）。升級後啟動時，Slurm 守護程式會將其相關狀態檔案和資料庫更新為新版本使用的結構。如果還原到舊版本，相關的 Slurm 守護程式將無法識別新的狀態檔案或資料庫，導致狀態資訊或作業計費遺失或損壞。

透過使用復原工具（如完整檔案備份、磁碟映像和快照），可能可以將元件還原到升級前的狀態。特別是，如果希望還原升級，需要還原 `StateSaveLocation` 的內容和計費資料庫。

---

### 維護版本升級

升級到較新的維護版本時，建議遵循與主要版本相同的升級程序。您會發現過程花費的時間較少，且更能容忍混合版本和原地降級。但是，您應該始終有最新的備份以鞏固您的復原選項。

---

### 升級程序

升級程序摘要如下。注意守護程式應升級的特定順序：

1. **準備**叢集進行升級
2. **建立備份**
3. 升級 **slurmdbd**
4. 升級 **slurmctld**
5. 升級 **slurmd**（最好與 slurmctld 一起）
6. 升級**登入節點和用戶端命令**
7. 重新編譯/升級**自訂 Slurm 外掛程式**
8. 測試關鍵功能
9. 封存備份資料

**注意**：如果使用 RPM/DEB 套件，每個系統上的所有套件必須一起升級。避免使用低階套件管理器如 `rpm` 或 `dpkg`，因為它們可能無法正確執行這些相依性。

---

### 準備

#### RELEASE_NOTES 和 CHANGELOG

審閱目標版本的 `RELEASE_NOTES.md` 檔案中的相關發行說明，以及您目前執行的版本和目標版本之間的任何主要版本。特別注意任何**移除**或**變更**的項目。

#### 設定變更

在升級生產環境之前，始終在測試環境中準備和測試設定變更。發行說明中概述的變更需要在 man pages（如 slurm.conf）中查詢詳細資訊和新語法。

如果啟用了 `SlurmctldParameters=reconfig_on_restart`，應格外小心，因為 slurmd 和 sackd 守護程式可能比預期更早升級。暫時停用該選項可能是明智的。

#### 計劃停機時間

參考以下章節中每個相關 Slurm 守護程式的預期停機時間指導，特別是 slurmdbd。通知受影響的使用者相關服務的估計停機時間及對其作業的潛在影響。

#### OpenAPI 變更

使用 `--json` 或 `--yaml` 參數的站台或執行 `slurmrestd` 的站台，需要在升級前檢查格式相容性和 data_parser 外掛程式移除。

---

### 建立備份

**始終**建立完整備份以還原 Slurm 的所有部分，包括 MySQL 資料庫。SchedMD 致力於使支援的升級成為無縫過程，但可能會出現意外問題並**不可逆轉地損壞** Slurm 保留的所有資料。

至少備份以下項目：

1. **StateSaveLocation**
   ```bash
   scontrol show config | grep StateSaveLocation
   ```

2. **整個 Slurm 設定目錄**（通常在 `/etc/slurm/`）

3. **MySQL 資料庫**（如果設定了 slurmdbd）
   ```bash
   # 建議在 slurmdbd 停止時執行
   mysqldump --databases slurm_acct_db > /path/to/backup.sql

   # 如果需要在執行中備份，可使用 --single-transaction
   # 但有以下限制：
   # - 備份期間資料庫操作可能較慢
   # - 還原時會遺失備份開始後的任何變更
   ```

---

### slurmdbd（計費）

如果您的環境使用 slurmdbd，它必須與 slurmctld 守護程式具有相同或更高的主要版本號，且版本需在相容性範圍內。因此，執行升級時應**首先升級 slurmdbd**。

slurmdbd 的升級可能需要顯著的**停機時間**。對於大型計費資料庫，預防性資料庫轉儲會花費一些時間，且升級後的守護程式在更新資料庫到新結構時可能會無回應數十分鐘。

建議的升級程序：

1. 正常關閉 slurmdbd 守護程式
   ```bash
   sacctmgr shutdown
   # 或
   systemctl stop slurmdbd
   ```

2. 備份 Slurm 資料庫

3. 驗證 my.cnf 中的 `innodb_buffer_pool_size` 大於預設值

4. 升級 slurmdbd 守護程式二進位檔、函式庫和 systemd 單元檔案

5. 啟動主要 slurmdbd 守護程式
   ```bash
   # 建議先以 SlurmUser 直接啟動
   sudo -u slurm slurmdbd -D
   # 完成後 Ctrl+C 結束，再用 systemd 啟動
   ```

6. 啟動備份 slurmdbd 守護程式（如適用）

7. 驗證計費操作

#### 資料庫伺服器

升級 slurmdbd 使用的資料庫伺服器（如 MySQL 或 MariaDB）時，通常不需要特殊程序。當從較舊版本的 MariaDB 或任何版本的 MySQL 升級到 **MariaDB 10.2.1** 或更高版本時，確保您執行的是 **slurmdbd 22.05.7** 或更高版本。

---

### slurmctld（控制器）

建議同時升級 slurmctld 系統和計算節點上的 slurmd 以及用戶端機器和登入節點上的其他 Slurm 命令。當使用多個 slurmctld 主機時，所有主機應同時升級。

升級 slurmctld 涉及短暫的**停機時間**，在此期間不接受作業提交，佇列中的作業不會被排程，完成中作業的資訊會被保留。一旦升級後的控制器啟動，這些功能將恢復。

建議的升級程序：

1. 增加設定的 SlurmdTimeout 和 SlurmctldTimeout 值
   ```bash
   scontrol reconfig
   ```

2. 關閉 slurmctld 守護程式

3. （選用）關閉計算節點上的 slurmd 守護程式

4. 備份設定的 StateSaveLocation 內容

5. 升級 slurmctld（和選用的 slurmd）守護程式及其 systemd 服務檔案

6. （選用）重新啟動計算節點上的 slurmd 守護程式

7. 重新啟動 slurmctld 守護程式

8. 驗證正常操作

9. 還原首選的 SlurmdTimeout 和 SlurmctldTimeout 值

---

### slurmd（計算節點）

建議同時升級所有 slurmd 節點和 slurmctld。也可以透過稍後以任意數量的群組升級 slurmd 節點來執行滾動升級。

只要在過程中未達到 **SlurmdTimeout**，升級不會中斷執行中的作業。但是，當 slurmd 因升級而停機時，新作業不會啟動，完成中的作業會等待向控制器報告直到它重新上線。

如果您單獨升級 slurmd 節點，可遵循以下程序：

1. 增加設定的 SlurmdTimeout 值並執行 `scontrol reconfig`
2. 關閉計算節點上的 slurmd 守護程式
3. 備份設定的 StateSaveLocation 內容
4. 升級 slurmd 守護程式及其 systemd 單元檔案
5. 重新啟動 slurmd 守護程式
6. 驗證正常操作
7. 對任何其他需要升級的節點群組重複
8. 還原首選的 SlurmdTimeout 值

---

### 其他 Slurm 命令

其他 Slurm 命令（包括用戶端命令）升級時不需要特別注意，除非發行說明中特別註明。在核心 Slurm 元件升級後，使用系統的正常方法升級其他元件及其 systemd 單元檔案和用戶端命令，然後重新啟動任何受影響的守護程式。

---

### 自訂 Slurm 外掛程式

Slurm 的主要公開 API 函式庫（libslurm.so.X.0.0）會在每個主要版本增加其版本號，因此任何連結到它的應用程式應在升級後重新編譯。這包括本地開發的 Slurm 外掛程式。

如果您建置了自己版本的 Slurm 外掛程式，除了必須重新編譯它們外，它們可能還需要修改以支援新版本的 Slurm。

PMI-1（libpmi.so.0.0.0）和 PMI-2（libpmi2.so.0.0.0）公開 API 函式庫在版本間不會變更，意味著連結到它們不需要在 Slurm 升級後重新編譯應用程式。

檢查應用程式是否需要重新編譯的簡單方法：
```bash
ldd /path/to/binary | grep slurm
# 如果看到版本化的 libslurm.so.x.y.z 參考，則需要重新編譯
```

---

### 無縫升級

在自訂 Slurm 建置過程的環境中，可以將新版本的 Slurm 安裝到唯一目錄，並使用符號連結將 PATH 中的目錄指向您想使用的 Slurm 版本。這允許您在維護期間之前安裝新版本，並在需要回滾時輕鬆切換版本。

---

## 說明

### 升級順序圖

```
1. slurmdbd（計費守護程式）
      ↓
2. slurmctld（控制器守護程式）
      ↓
3. slurmd（計算節點守護程式）
      ↓
4. 其他命令和外掛程式
```

### 版本相容性視覺化

```
24.11 可接受: ←── 24.05 ←── 23.11 ←── 23.02
              |
              ↓
         目前版本
```

---

## 實務範例

### 完整升級腳本範例

```bash
#!/bin/bash
# Slurm 升級腳本範例

# 1. 增加超時時間
scontrol update slurmdtimeout=1800 slurmctldtimeout=1800
scontrol reconfig

# 2. 停止 slurmdbd
systemctl stop slurmdbd

# 3. 備份資料庫
mysqldump --databases slurm_acct_db > /backup/slurm_db_$(date +%Y%m%d).sql

# 4. 備份狀態目錄
STATE_DIR=$(scontrol show config | grep StateSaveLocation | awk '{print $3}')
cp -a $STATE_DIR /backup/state_$(date +%Y%m%d)

# 5. 升級套件（以 RPM 為例）
yum upgrade slurm*

# 6. 啟動 slurmdbd
sudo -u slurm slurmdbd -D &
sleep 60  # 等待資料庫更新
pkill slurmdbd
systemctl start slurmdbd

# 7. 啟動 slurmctld
systemctl start slurmctld

# 8. 在計算節點上升級並啟動 slurmd
# pdsh -a "yum upgrade slurm* && systemctl restart slurmd"

# 9. 還原超時時間
scontrol update slurmdtimeout=300 slurmctldtimeout=300
scontrol reconfig

echo "升級完成"
```

### 驗證升級

```bash
# 檢查版本
sinfo --version
scontrol show config | grep SLURM_VERSION

# 檢查守護程式狀態
scontrol ping
sacctmgr show cluster

# 測試作業提交
srun -N1 hostname
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未按順序升級守護程式 | 始終按 slurmdbd → slurmctld → slurmd 順序 |
| 未備份就升級 | 始終先備份資料庫和 StateSaveLocation |
| 嘗試降級 | 降級不受支援，需從備份還原 |
| 跳過不相容版本 | 必須逐步升級到相容版本 |
| 未增加超時時間 | 升級前增加 SlurmdTimeout 和 SlurmctldTimeout |
| 使用 EPEL 的 Slurm | 在 EPEL 設定中排除 slurm* 套件 |

### 升級前檢查清單

- [ ] 審閱 RELEASE_NOTES
- [ ] 確認版本相容性
- [ ] 備份 MySQL 資料庫
- [ ] 備份 StateSaveLocation
- [ ] 備份設定檔目錄
- [ ] 增加超時時間值
- [ ] 通知使用者計劃停機
- [ ] 準備回滾計劃

---

## 快速參考

### 升級順序

```
slurmdbd → slurmctld → slurmd → 其他命令
```

### 重要備份位置

| 項目 | 位置 |
|------|------|
| 資料庫 | `mysqldump --databases slurm_acct_db` |
| 狀態檔案 | StateSaveLocation（slurm.conf 中定義）|
| 設定檔 | 通常在 `/etc/slurm/` |

### 常用命令

| 命令 | 功能 |
|------|------|
| `sinfo --version` | 顯示版本 |
| `scontrol ping` | 測試控制器連線 |
| `sacctmgr shutdown` | 正常關閉 slurmdbd |
| `scontrol reconfig` | 重新載入設定 |

### 相關文件

- [管理員快速入門](quickstart_admin.md)
- [計費設定](accounting.md)
- [故障排除](troubleshoot.md)
