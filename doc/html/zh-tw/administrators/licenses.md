# Slurm 授權管理指南 (Licenses Guide)

## TL;DR

Slurm 可在排程時將可用授權分配給作業，協助軟體授權管理。授權不可用時作業會保持等待。授權分為本地授權（slurm.conf 設定）和遠端授權（sacctmgr 設定，由資料庫提供）。23.02+ 支援動態授權追蹤與第三方授權伺服器整合。

---

## 翻譯

### 授權概觀

Slurm 可以透過在排程時將可用授權分配給作業來協助軟體授權管理。如果授權不可用，作業會保持等待直到授權變為可用。Slurm 中的授權本質上是共用資源，意味著設定的資源不綁定到特定主機，而是與整個叢集關聯。

Slurm 中的授權可以透過兩種方式設定：

- **本地授權 (Local Licenses)**：本地授權是在設定它們的 slurm.conf 中的叢集本地使用。
- **遠端授權 (Remote Licenses)**：遠端授權由資料庫提供，使用 sacctmgr 命令設定。遠端授權本質上是動態的，因為在執行 sacctmgr 命令時，slurmdbd 會更新分配授權的所有叢集。

---

### 本地授權

本地授權在 slurm.conf 中使用 `Licenses` 選項定義。

**slurm.conf：**
```
Licenses=fluent:30,ansys:100
```

**查看設定的授權：**
```bash
$ scontrol show lic
LicenseName=ansys
    Total=100 Used=0 Free=100 Remote=no
LicenseName=fluent
    Total=30 Used=0 Free=30 Remote=no
```

**請求授權：**

使用 `-L` 或 `--licenses` 提交選項：
```bash
$ sbatch -L ansys:2 script.sh
Submitted batch job 5212

$ scontrol show lic
LicenseName=ansys
    Total=100 Used=2 Free=98 Remote=no
```

**使用 --tres-per-task 請求授權：**

如果使用此方法，授權也必須在 slurm.conf 的 `AccountingStorageTRES` 選項中定義。

**slurm.conf：**
```
Licenses=fluent:30
AccountingStorageTRES=license/fluent
```

```bash
$ sbatch --tres-per-task=license/fluent:4 script.sh
Submitted batch job 6482
```

---

### 遠端授權

遠端授權本身**不**提供與第三方授權管理器的任何整合。在建立這些授權時使用 "Server" 和 "ServerType" 參數僅供參考，並不意味著與這些伺服器有任何自動授權管理。系統管理員負責實現與這些系統所需的任何整合。

#### 使用案例

一個站台有兩台授權伺服器，一台提供 100 個 FlexNet 的 Nastran 授權，另一台提供 50 個 Reprise License Management 的 Matlab 授權。站台有兩個叢集「fluid」和「pdf」用於執行使用這兩種產品的模擬作業。管理者想要將 Nastran 授權平均分配給兩個叢集，但將 70% 的 Matlab 授權分配給叢集「pdf」，剩餘 30% 分配給叢集「fluid」。

#### 設定 Slurm

假設兩個叢集已使用 sacctmgr 命令在 slurmdbd 中正確設定：

```bash
$ sacctmgr show clusters format=cluster,controlhost
   Cluster     ControlHost
---------- ---------------
     fluid     143.11.1.3
       pdf     144.12.3.2
```

**一步新增授權：**
```bash
$ sacctmgr add resource name=nastran cluster=fluid,pdf \
  count=100 allowed=50 server=flex_host servertype=flexlm type=license
 Adding Resource(s)
  nastran@flex_host
   Cluster - fluid	50
   Cluster - pdf	50
```

**多步新增授權：**
```bash
# 新增資源
$ sacctmgr add resource name=matlab count=50 server=rlm_host \
  servertype=rlm type=license

# 分配給叢集 pdf（70%）
$ sacctmgr add resource name=matlab server=rlm_host \
  cluster=pdf allowed=70

# 分配給叢集 fluid（30%）
$ sacctmgr add resource name=matlab server=rlm_host \
  cluster=fluid allowed=30
```

**查看授權：**
```bash
$ sacctmgr show resource withclusters
      Name     Server     Type  Count LastConsumed Allocated ServerType    Cluster  Allowed
---------- ---------- -------- ------ ------------ --------- ---------- ---------- --------
   nastran  flex_host  License    100            0       100     flexlm      fluid       50
   nastran  flex_host  License    100            0       100     flexlm        pdf       50
    matlab   rlm_host  License     50            0       100        rlm      fluid       30
    matlab   rlm_host  License     50            0       100        rlm        pdf       70
```

**在叢集上查看：**
```bash
# 在叢集 "pdf" 上：
$ scontrol show lic
LicenseName=matlab@rlm_host
    Total=35 Used=0 Free=35 Reserved=0 Remote=yes
LicenseName=nastran@flex_host
    Total=50 Used=0 Free=50 Reserved=0 Remote=yes
```

**提交作業到遠端授權：**
```bash
$ sbatch -L nastran@flex_host script.sh
Submitted batch job 5172
```

**修改授權：**
```bash
# 修改總數
$ sacctmgr modify resource name=matlab server=rlm_host set count=200

# 修改叢集分配百分比
$ sacctmgr modify resource name=matlab server=rlm_host \
  cluster=pdf set allowed=60
```

**刪除授權：**
```bash
# 從特定叢集刪除
$ sacctmgr delete resource where name=matlab server=rlm_host cluster=fluid

# 完全刪除
$ sacctmgr delete resource where name=nastran server=flex_host
```

#### Absolute 旗標（23.02+）

從 Slurm 23.02 開始，新的 `Absolute` 旗標表示每個叢集的授權允許值應被視為絕對授權數量，而非百分比。

```bash
$ sacctmgr -i add resource name=deluxe cluster=fluid,pdf count=150 allowed=70 \
  server=flex_host servertype=flexlm flags=absolute

$ sacctmgr show resource withclusters
      Name     Server     Type  Count LastConsumed Allocated ServerType    Cluster  Allowed                Flags
---------- ---------- -------- ------ ------------ --------- ---------- ---------- -------- --------------------
    deluxe  flex_host  License    150            0       140     flexlm      fluid       70             Absolute
    deluxe  flex_host  License    150            0       140     flexlm        pdf       70             Absolute
```

可在 slurmdbd.conf 中新增 `AllResourcesAbsolute=yes` 將此設為所有新建授權的預設值。

---

### 動態授權（23.02+）

從 Slurm 23.02 開始，遠端授權的 `LastConsumed` 欄位設計為定期更新來自授權伺服器的活動使用計數。

**FlexLM lmstat 範例腳本：**
```bash
#!/bin/bash

set -euxo pipefail

LMSTAT=/opt/foobar/bin/lmstat
LICENSE=foobar

consumed=$(${LMSTAT} | grep "Users of ${LICENSE}"|sed "s/.*Total of \([0-9]\+\) licenses in use)/\1/")

sacctmgr -i update resource ${LICENSE} set lastconsumed=${consumed}
```

當透過 sacctmgr 變更 LastConsumed 值時，更新會自動推送到 Slurm 控制器。它們會使用此值計算 `LastDeficit` 值 — 此值表示從叢集角度「失蹤」了多少授權，需要暫時保留。

**範例：**

叢集有 100 個「foobar」授權，在「blackhole」叢集上分配 80 個：

```bash
$ sacctmgr add resource foobar count=100 flags=absolute cluster=blackhole allowed=80

$ scontrol show license
LicenseName=foobar@slurmdb
    Total=80 Used=0 Free=80 Reserved=0 Remote=yes
    LastConsumed=0 LastDeficit=0 LastUpdate=2023-02-28T16:36:55
```

更新 LastConsumed 為 30（外部使用）：

```bash
$ sacctmgr -i update resource foobar set lastconsumed=30

$ scontrol show license
LicenseName=foobar@slurmdb
    Total=80 Used=0 Free=70 Reserved=0 Remote=yes
    LastConsumed=30 LastDeficit=10 LastUpdate=2023-02-28T16:39:27
```

叢集計算出 10 個授權的赤字，只會排程最多 70 個授權。

---

## 說明

### 本地 vs 遠端授權

| 特性 | 本地授權 | 遠端授權 |
|------|----------|----------|
| 設定位置 | slurm.conf | sacctmgr |
| 作用範圍 | 單一叢集 | 多叢集 |
| 動態更新 | 需要重新設定 | 即時更新 |
| 第三方整合 | 無 | 可透過 LastConsumed |

### 授權分配流程

```
授權伺服器（FlexLM、RLM 等）
            ↓
    cron 腳本更新 LastConsumed
            ↓
        slurmdbd
            ↓
    ├── 叢集 A（分配 50%）
    └── 叢集 B（分配 50%）
```

---

## 實務範例

### 設定商業軟體授權

```bash
# 1. 定義授權資源
sacctmgr add resource name=matlab count=100 server=license_server \
  servertype=flexlm type=license

# 2. 分配給叢集
sacctmgr add resource name=matlab server=license_server \
  cluster=production allowed=80
sacctmgr add resource name=matlab server=license_server \
  cluster=development allowed=20

# 3. 設定動態追蹤（crontab）
*/5 * * * * /etc/slurm/scripts/update_matlab_licenses.sh
```

### 提交需要授權的作業

```bash
# 使用 -L 選項
sbatch -L matlab@license_server:4 simulation.sh

# 使用 --tres-per-task（需要在 slurm.conf 中設定）
sbatch --tres-per-task=license/matlab:4 simulation.sh

# 檢查授權使用狀態
scontrol show lic
squeue -o "%.10i %.10L"
```

### 監控授權使用

```bash
# 查看所有授權
scontrol show lic

# 查看詳細資源資訊
sacctmgr show resource withclusters

# 查看作業使用的授權
squeue -o "%.10i %.10u %.10L"
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 授權總分配超過 100% | 使用 Absolute 旗標或確保百分比總和合理 |
| 未同步 LastConsumed | 設定 cron 定期更新外部授權使用量 |
| 遠端授權格式錯誤 | 使用 name@server 格式請求遠端授權 |
| 忽略 LastDeficit | 監控赤字，了解授權分配問題 |

### 除錯技巧

```bash
# 檢查授權設定
scontrol show lic

# 查看為何作業等待授權
squeue -o "%.10i %.10P %.20j %.8T %.10M %R" | grep License

# 檢查資料庫中的授權
sacctmgr show resource withclusters

# 測試授權請求
srun -L matlab@server:1 hostname
```

---

## 快速參考

### slurm.conf 設定

```
# 本地授權
Licenses=fluent:30,ansys:100

# TRES 追蹤（可選）
AccountingStorageTRES=license/fluent,license/ansys
```

### sacctmgr 命令

| 命令 | 功能 |
|------|------|
| `sacctmgr add resource name=X count=N ...` | 新增授權資源 |
| `sacctmgr modify resource name=X set ...` | 修改授權 |
| `sacctmgr delete resource where name=X` | 刪除授權 |
| `sacctmgr show resource withclusters` | 顯示授權分配 |

### 作業提交選項

| 選項 | 說明 | 範例 |
|------|------|------|
| `-L, --licenses` | 請求授權 | `-L ansys:2` |
| `--tres-per-task` | 每任務 TRES | `--tres-per-task=license/fluent:1` |

### 授權資源欄位

| 欄位 | 說明 |
|------|------|
| Count | 授權總數 |
| Allowed | 叢集分配（百分比或絕對數） |
| LastConsumed | 外部使用量 |
| LastDeficit | 赤字（外部使用超出預期） |

### 相關文件

- [計費設定](accounting.md)
- [資源限制](resource_limits.md)
- [TRES](tres.md)
