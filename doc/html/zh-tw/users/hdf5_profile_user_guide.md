# Slurm 使用 HDF5 進行效能剖析使用者指南 (Profiling Using HDF5)

## TL;DR

Slurm 的 acct_gather_profile/hdf5 外掛程式允許收集比資料庫更詳細的作業效能資料。可收集的時間序列資料包括：能源消耗、網路 I/O（InfiniBand）、檔案系統 I/O（Lustre）和任務效能（CPU、記憶體、磁碟）。使用 `--profile` 選項啟用，資料儲存為 HDF5 格式，可用 `sh5util` 工具合併和提取。

---

## 翻譯

### 目錄

- 概觀
- 管理
- 作業剖析
- HDF5
- 資料結構

---

### 概觀

acct_gather_profile/hdf5 外掛程式允許 Slurm 協調收集叢集上執行作業的資料，這些資料比實際包含在其資料庫中的更為詳細。資料來自定期採樣各種效能資料，這些資料由 Slurm、作業系統或元件軟體收集。外掛程式會將每個來源的資料記錄為**時間序列**，並累積作業的每個統計資料的總計。

時間序列包括：由 acct_gather_energy 外掛程式收集的能源資料、由 acct_gather_interconnect 外掛程式收集的網路介面 I/O 資料、由 acct_gather_filesystem 外掛程式收集的平行檔案系統（如 Lustre）I/O 資料，以及由 jobacct_gather 外掛程式收集的任務效能資料（如本地磁碟 I/O、CPU 消耗和記憶體使用）。未來可能會新增其他來源的資料。

資料會收集到共享檔案系統上的檔案中，每個作業的每個分配節點上的每個步驟都有一個檔案，然後合併成一個 HDF5 檔案。選擇共享檔案系統上的個別檔案是因為資料可能非常龐大，因此透過 RPC 將資料傳遞給 Slurm 控制守護程式的解決方案可能無法擴展到非常大的叢集或具有許多分配節點的作業。

---

### 管理

#### 共享檔案系統

HDF5 Profile 外掛程式需要所有計算節點上有一個共同的共享檔案系統。作業執行時，外掛程式會為每個節點上作業的每個步驟將檔案寫入此檔案系統。作業結束時，會啟動合併程序，將節點-步驟檔案合併成一個作業的 HDF5 檔案。

目錄結構的根目錄在 acct_gather.conf 檔案中的 **ProfileHDF5Dir** 選項中宣告。如果目錄不存在，Slurm 會建立它。每個使用者在 ProfileHDF5Dir 中都有自己的目錄，其中包含 HDF5 檔案。所有目錄和檔案都由 SlurmdUser（通常是 root）建立。使用者特定的目錄及其中的檔案會被 chown 給執行作業的使用者，以便他們可以存取檔案。由於通常是 root 使用者建立這些檔案/目錄，因此 root squashed 檔案系統不適用於 ProfileHDF5Dir。

每個建立剖析檔的使用者在剖析目錄中都會有一個子目錄，該目錄僅對該使用者具有讀寫權限。

#### 設定參數

剖析外掛程式在 slurm.conf 檔案中啟用，並在 acct_gather.conf 檔案中進行內部設定。

**slurm.conf 參數：**

| 參數 | 說明 |
|------|------|
| `AcctGatherProfileType=acct_gather_profile/hdf5` | 啟用 HDF5 外掛程式 |
| `JobAcctGatherFrequency=<seconds>` | 設定資料類型的採樣頻率 |

**acct_gather.conf 參數：**

| 參數 | 說明 |
|------|------|
| `ProfileHDF5Dir=<path>` | acct_gather_profile 外掛程式將詳細資料作為 HDF5 檔案寫入的共享資料夾路徑。這是必要參數 |
| `ProfileHDF5Default=[options]` | 為每個作業提交收集的資料類型的逗號分隔列表。謹慎使用此選項 |

#### 時間序列控制參數

其他外掛程式將時間序列資料新增到 HDF5 收集中。它們通常在 slurm.conf 的 JobAcctGatherFrequency 參數中指定預設輪詢頻率。輪詢頻率可以使用 srun 的 `--acctg-freq` 參數覆蓋。兩者都是 `task=sec,energy=sec,filesystem=sec,network=sec` 的格式。

IPMI 能源外掛程式還需要在 acct_gather.conf 檔案中設定 EnergyIPMIFrequency 值。這設定外掛程式採樣外部感測器的速率。此值應與 JobAcctGatherFrequency 或 --acctg-freq 中的 energy=sec 相同。

**注意**：IPMI 和剖析採樣不是同步的。剖析樣本只是取最後可用的 IPMI 樣本值。如果剖析能源樣本比 IPMI 樣本速率更頻繁，IPMI 值將被重複。如果剖析能源樣本大於 IPMI 速率，IPMI 值將會遺失。

---

### 作業剖析

#### 資料收集

salloc|sbatch|srun 上的 `--profile` 選項控制是否收集資料以及收集什麼類型的資料。如果未指定 --profile，則不收集資料，除非在 acct_gather.conf 中使用了 **ProfileHDF5Default** 選項。命令列上的 --profile 會覆蓋設定檔中指定的任何值。

```
--profile=<all|none|[energy[,|task[,|filesystem[,|network]]]]>
```

| 選項 | 說明 |
|------|------|
| `all` | 收集所有資料類型（不能與其他值組合） |
| `none` | 不收集任何資料類型。這是預設值（不能與其他值組合） |
| `energy` | 收集能源資料 |
| `filesystem` | 收集檔案系統資料。目前僅支援 Lustre 檔案系統 |
| `network` | 收集網路（InfiniBand）資料 |
| `task` | 收集任務（I/O、記憶體等）資料 |

#### 資料合併

節點-步驟檔案使用 sh5util 合併成一個作業的 HDF5 檔案。

如果作業是用 sbatch 啟動的，可以將命令列加到正常的啟動腳本中，例如：

```bash
sbatch -n1 -d$SLURM_JOB_ID --wrap="sh5util -j $SLURM_JOB_ID"
```

#### 資料提取

sh5util 程式也可用於從 HDF5 檔案中提取特定資料，並以逗號分隔值（csv）格式寫入，以便匯入其他分析工具（如試算表）。

---

### HDF5

HDF5 是一種著名的結構化資料集，允許將異質但相關的資料儲存在一個檔案中（例如能源統計、網路 I/O、任務資料等部分）。其內部結構類似於檔案系統，**groups** 類似於目錄，**data sets** 類似於檔案。它還允許將 **attributes** 附加到 groups 以儲存應用程式定義的屬性。

有現成的程式（特別是 HDFView）可用於查看和操作這些檔案。

---

### 資料結構

在作業檔案中，作業的每個**步驟**都會有一個群組。在每個步驟中，會有一個節點群組和一個任務群組。

- **nodes** 群組對步驟分配中的每個節點都有一個群組。對於每個節點群組，有一個用於時間序列的子群組和另一個用於總計的子群組。
  - **Time Series** 群組包含每個收集器的時間序列群組/資料集
  - **Totals** 群組包含時間序列中每個項目的最小值、平均值、最大值和總和

- **Tasks** 群組只包含每個任務的子群組。它主要包含一個屬性，說明任務在哪個節點上執行。這組群組本質上是一個交叉參照表。

#### 能源資料

需要在 slurm.conf 中設定 `AcctGatherEnergyType=acct_gather_energy/ipmi` 才能收集能源資料。

每個能源時間序列中的資料樣本包含以下資料項目：

| 項目 | 說明 |
|------|------|
| Date Time | 取得資料樣本的時間。可用於與其他來源（如日誌）關聯活動 |
| Time | 自步驟開始以來的經過時間 |
| Power | 間隔期間的功耗 |
| CPU Frequency | 取樣時的 CPU 頻率（千赫茲） |

#### 檔案系統資料

需要在 slurm.conf 中設定 `AcctGatherFilesystemType=acct_gather_filesystem/lustre` 才能收集檔案系統資料。

每個檔案系統時間序列中的資料樣本包含以下資料項目：

| 項目 | 說明 |
|------|------|
| Date Time | 取得資料樣本的時間 |
| Time | 自步驟開始以來的經過時間 |
| Reads | 讀取操作次數 |
| Megabytes Read | 讀取的 MB 數 |
| Writes | 寫入操作次數 |
| Megabytes Write | 寫入的 MB 數 |

#### 網路資料（InfiniBand）

需要在 slurm.conf 中設定 `AcctGatherInterconnectType=acct_gather_interconnect/ofed` 才能收集網路資料。

每個網路時間序列中的資料樣本包含以下資料項目：

| 項目 | 說明 |
|------|------|
| Date Time | 取得資料樣本的時間 |
| Time | 自步驟開始以來的經過時間 |
| Packets In | 傳入的封包數 |
| Megabytes Read | 透過介面傳入的 MB 數 |
| Packets Out | 傳出的封包數 |
| Megabytes Write | 透過介面傳出的 MB 數 |

#### 任務資料

需要在 slurm.conf 中設定 `JobAcctGatherType=jobacct_gather/linux` 才能收集任務資料。

每個任務時間序列中的資料樣本包含以下資料項目：

| 項目 | 說明 |
|------|------|
| Date Time | 取得資料樣本的時間 |
| Time | 自步驟開始以來的經過時間 |
| CPU Frequency | 取樣時的 CPU 頻率 |
| CPU Time | 樣本期間使用的 CPU 時間秒數 |
| CPU Utilization | 間隔期間的 CPU 利用率 |
| RSS | 取樣時的 RSS 值 |
| VM Size | 取樣時的虛擬記憶體大小值 |
| Pages | 樣本中使用的頁面數 |
| Read Megabytes | 從本地磁碟讀取的 MB 數 |
| Write Megabytes | 寫入本地磁碟的 MB 數 |

---

## 說明

### HDF5 檔案結構

```
Job_<jobid>.h5
├── Step_0/
│   ├── Nodes/
│   │   ├── node001/
│   │   │   ├── Time_Series/
│   │   │   │   ├── Energy/
│   │   │   │   ├── Filesystem/
│   │   │   │   ├── Network/
│   │   │   │   └── Task/
│   │   │   └── Totals/
│   │   │       ├── Energy/
│   │   │       ├── Filesystem/
│   │   │       ├── Network/
│   │   │       └── Task/
│   │   └── node002/
│   │       └── ...
│   └── Tasks/
│       ├── Task_0/
│       ├── Task_1/
│       └── ...
└── Step_1/
    └── ...
```

### 資料類型與外掛程式對應

| 資料類型 | slurm.conf 外掛程式 | --profile 選項 |
|----------|---------------------|----------------|
| 能源 | AcctGatherEnergyType=acct_gather_energy/ipmi | energy |
| 檔案系統 | AcctGatherFilesystemType=acct_gather_filesystem/lustre | filesystem |
| 網路 | AcctGatherInterconnectType=acct_gather_interconnect/ofed | network |
| 任務 | JobAcctGatherType=jobacct_gather/linux | task |

---

## 實務範例

### 啟用剖析提交作業

```bash
# 收集所有資料類型
sbatch --profile=all my_script.sh

# 只收集能源和任務資料
sbatch --profile=energy,task my_script.sh

# 收集任務和檔案系統資料
srun --profile=task,filesystem ./my_program
```

### 設定採樣頻率

```bash
# 使用預設頻率
sbatch --profile=all my_script.sh

# 自訂採樣頻率
srun --profile=all --acctg-freq=task=30,energy=30 ./my_program
```

### 合併剖析資料

```bash
# 作業完成後合併
sh5util -j 12345

# 在作業腳本中自動合併
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --profile=all

srun ./my_program

# 提交合併作業
sbatch -n1 -d$SLURM_JOB_ID --wrap="sh5util -j $SLURM_JOB_ID"
```

### 提取資料為 CSV

```bash
# 提取特定資料
sh5util -j 12345 -E -l Node:TimeSeries -s Energy -o energy_data.csv

# 提取任務資料
sh5util -j 12345 -E -l Node:TimeSeries -s Task -o task_data.csv
```

### 使用 HDFView 查看資料

1. 下載並安裝 HDFView
2. 開啟 HDF5 檔案
3. 瀏覽樹狀結構查看群組和資料集
4. 雙擊資料集查看內容

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 共享檔案系統不可用 | 確保 ProfileHDF5Dir 在所有節點都可存取 |
| 使用 root squashed 檔案系統 | ProfileHDF5Dir 不能使用 root squashed 檔案系統 |
| 未設定必要的外掛程式 | 確保在 slurm.conf 中設定相應的 AcctGather*Type |
| 採樣頻率不一致 | 確保 IPMI 頻率與剖析頻率匹配 |

### 設定檢查清單

1. ✅ slurm.conf 中設定 `AcctGatherProfileType=acct_gather_profile/hdf5`
2. ✅ acct_gather.conf 中設定 `ProfileHDF5Dir=<path>`
3. ✅ ProfileHDF5Dir 是共享檔案系統且不是 root squashed
4. ✅ 根據需要收集的資料類型設定相應的 AcctGather*Type
5. ✅ 設定適當的 JobAcctGatherFrequency

### 效能考量

- 較高的採樣頻率會產生更多資料
- 對於大型作業，HDF5 檔案可能會非常大
- 考慮使用 ProfileHDF5Default 進行測試環境而非生產環境

---

## 快速參考

### --profile 選項值

| 值 | 說明 |
|----|------|
| `all` | 收集所有資料類型 |
| `none` | 不收集任何資料類型 |
| `energy` | 能源資料 |
| `filesystem` | 檔案系統 I/O 資料 |
| `network` | 網路 I/O 資料 |
| `task` | 任務效能資料 |

### sh5util 常用選項

| 選項 | 說明 |
|------|------|
| `-j <jobid>` | 指定作業 ID |
| `-E` | 提取資料為 CSV |
| `-l <level>` | 指定資料層級 |
| `-s <series>` | 指定時間序列類型 |
| `-o <file>` | 輸出檔案名稱 |

### 關鍵設定檔

| 檔案 | 相關參數 |
|------|----------|
| slurm.conf | AcctGatherProfileType, JobAcctGatherFrequency, AcctGatherEnergyType, AcctGatherFilesystemType, AcctGatherInterconnectType, JobAcctGatherType |
| acct_gather.conf | ProfileHDF5Dir, ProfileHDF5Default, EnergyIPMIFrequency |
