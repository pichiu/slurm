# Slurm 多類別安全 (MCS) 指南

## TL;DR

MCS (Multi-Category Security) 外掛程式擴展作業節點**獨佔性**和作業/節點資訊**隱私**功能。MCS 標籤可基於帳戶 (mcs/account)、群組 (mcs/group)、使用者 (mcs/user) 或任意字串 (mcs/label)。設定 `MCSPlugin` 和 `MCSParameters` 控制標籤強制、節點選擇過濾和隱私資料。使用 `--mcs-label` 和 `--exclusive=mcs` 選項。

---

## 翻譯

### 概觀

MCS 外掛程式旨在擴展目前 Slurm 與作業節點**獨佔性**和作業/節點資訊顯示**隱私**相關的功能。

#### 獨佔性擴展

Slurm 的 `OverSubscribe` 選項控制分割區是否能在每個資源上同時執行多個作業，無論「作業類型」為何。作業提交客戶端也可使用 `--exclusive` 和 `--oversubscribe` 參數請求作業如何[共享](cons_tres_share.md)。

`ExclusiveUser` slurm.conf 參數和 `--exclusive=user` 客戶端參數值修改獨佔功能。啟用此參數後，在考慮獨佔性時「作業類型」變得重要，因此作業可以**基於**作業使用者共享資源，意味著只有相同使用者的作業才能共享資源。

透過 MCS 外掛程式，Slurm 現在可以設定將 **MCS_label** 關聯到作業，並選擇性確保節點只能在具有相同標籤的作業之間共享。這為 Slurm 管理獨佔性的方式提供了更多自由度。

#### 隱私擴展

Slurm 的 `PrivateData` slurm.conf 參數用於控制對一般使用者隱藏什麼類型的資訊。類似於獨佔性屬性，MCS 外掛程式也透過根據使用者對其 **MCS_label** 的存取權限過濾作業和/或節點資訊來擴展**隱私**。

這意味著隱私現在不那麼限制性，資訊不再只是對一般使用者隱藏或不隱藏，而是根據這些可設定/可請求的標籤與 PrivateData 選項協調進行**過濾**。

---

### 限制

使用 MCS 會限制[搶佔](preempt.md)的操作。具體而言，任何具有 MCS 標籤並根據該標籤請求節點獨佔的作業，都將被阻止搶佔或被不匹配該標籤的任何作業搶佔。

如果設定了 `MCSParameters=enforced,select`，這些限制將適用於所有作業。

---

### 設定

兩個參數可用於設定 MCS：**MCSPlugin** 和 **MCSParameters**。

#### MCSPlugin

指定應使用哪個外掛程式。外掛程式是互斥的，要關聯的標籤類型取決於載入的外掛程式。

| 外掛程式 | 說明 |
|----------|------|
| `mcs/none` | 預設值，停用 MCS 標籤和功能 |
| `mcs/account` | MCS 標籤只能等於作業的 --account（需啟用計費）|
| `mcs/group` | MCS 標籤只能等於作業使用者的群組 |
| `mcs/user` | MCS 標籤只能等於作業 --uid 的使用者名稱 |
| `mcs/label` | MCS 標籤是任意字串（建議使用 job_submit 外掛程式管理）|

#### 查看 MCS 標籤

| 命令 | 說明 |
|------|------|
| `squeue -o "%i %u %M"` 或 `--format=mcslabel` | 顯示作業的 MCS 標籤 |
| `scontrol show job` | 顯示作業詳細資訊包含 MCS 標籤 |
| `scontrol show nodes` | 顯示節點的 MCS 標籤（從分配作業繼承）|
| `sview` | 圖形介面查看 MCS 標籤 |

#### 更新 MCS 標籤

```bash
scontrol update job <jobid> MCSLabel=<new_label>
```

**注意**：只有 PENDING 狀態的作業可以修改。

---

#### MCSParameters

指定傳遞給特定 MCS 外掛程式實作的選項。格式：

```
[ondemand|enforced][,noselect|select|ondemandselect][,privatedata]:[mcs_plugin_parameters]
```

預設值：`ondemand,ondemandselect`，無 privatedata。

##### 標籤強制選項

| 選項 | 說明 |
|------|------|
| `ondemand` | 僅在使用 --mcs-label 選項時設定 MCS 標籤 |
| `enforced` | 總是設定 MCS 標籤（強制）|

##### 節點選擇過濾選項

| 選項 | 說明 |
|------|------|
| `noselect` | 永不根據 MCS 標籤過濾節點 |
| `select` | 總是根據 MCS 標籤過濾節點 |
| `ondemandselect` | 僅在使用 --exclusive=mcs 時過濾（預設）|

##### 隱私資料選項

| 設定 | 說明 |
|------|------|
| `privatedata` + `PrivateData=jobs` | 根據 MCS 標籤過濾作業資訊 |
| `privatedata` + `PrivateData=nodes` | 根據 MCS 標籤過濾節點資訊 |

**警告**：使用 mcs/label 搭配 privatedata 會停用大部分 `PrivateData=[jobs|nodes]` 提供的過濾，因為假設所有使用者都有權存取所有標籤。

##### 外掛程式特定參數

目前只有 mcs/group 支援此選項，可用於指定允許被 mcs/group 外掛程式對應到 MCS 標籤的使用者群組清單（以 `|` 字元分隔）。

---

### 行為對照表

| 節點過濾 | 標籤強制：ondemand | 標籤強制：enforced |
|----------|-------------------|-------------------|
| **noselect** | 即使請求 --exclusive=mcs 也不過濾節點 | 即使請求 --exclusive=mcs 也不過濾節點 |
| **select** | **僅當**作業 MCS_label 已設定時過濾節點 | 總是過濾節點 |
| **ondemandselect** | **僅當** --exclusive=mcs 時過濾節點 | **僅當** --exclusive=mcs 時過濾節點 |

---

### 範例

#### mcs/account 設定

```
# slurm.conf
MCSPlugin=mcs/account
MCSParameters=enforced,select,privatedata
```

#### mcs/group 設定

```
# slurm.conf
MCSPlugin=mcs/group
MCSParameters=ondemand,noselect:groupA|groupB|groupC
```

#### mcs/user 設定

```
# slurm.conf
MCSPlugin=mcs/user
MCSParameters=enforced,select,privatedata
```

#### mcs/label 設定

```
# slurm.conf
MCSPlugin=mcs/label
MCSParameters=ondemand,select
```

---

### 使用範例

#### 查看 MCS 參數

```bash
scontrol show config | grep MCS
MCSPlugin          = mcs/group
MCSParameters      = ondemand,noselect:groupA|groupB|groupC
```

#### 在作業上設定 MCS 標籤

```bash
srun -n10 -t 1000 --mcs-label=groupB ./job &
```

#### 設定 MCS 標籤並啟用獨佔

```bash
srun -n10 -t 1000 --mcs-label=groupB --exclusive=mcs ./job &
```

#### 使用 mcs/account 設定不同標籤

```bash
srun -n10 -t 1000 -A another_account_than_default ./job &
```

#### 查看作業的 MCS 標籤

```bash
squeue -O jobid,username,mcslabel
JOBID               USER                MCSLABEL
2                   foo                 groupA
3                   bar                 groupB
```

#### 查看節點的 MCS 標籤

```bash
scontrol show nodes
NodeName=node0001 ...
   State=IDLE ... MCS_label=groupA
```

---

## 說明

### MCS 外掛程式選擇指南

| 需求 | 建議外掛程式 |
|------|--------------|
| 按計費帳戶隔離 | mcs/account |
| 按 Unix 群組隔離 | mcs/group |
| 按使用者隔離 | mcs/user |
| 自訂標籤系統 | mcs/label |

### 使用情境

```
mcs/account:
  - 部門按帳戶計費
  - 同帳戶的作業可共享節點

mcs/group:
  - 專案群組共享資源
  - 限制特定群組可用的標籤

mcs/user:
  - 每個使用者有自己的資源空間
  - 類似 ExclusiveUser 但更有彈性

mcs/label:
  - 完全自訂的標籤系統
  - 需要 job_submit 外掛程式管理
```

---

## 實務範例

### 按帳戶隔離（強制模式）

```
# slurm.conf
MCSPlugin=mcs/account
MCSParameters=enforced,select,privatedata
PrivateData=jobs,nodes
```

此設定：
- 每個作業必須有 MCS 標籤（= 帳戶名稱）
- 節點只與相同帳戶的作業共享
- 使用者只能看到自己帳戶的作業和節點

### 按群組隔離（隨選模式）

```
# slurm.conf
MCSPlugin=mcs/group
MCSParameters=ondemand,ondemandselect:research|teaching|admin
```

使用：
```bash
# 使用群組標籤並請求獨佔
srun --mcs-label=research --exclusive=mcs ./app
```

### 多租戶環境

```
# slurm.conf
MCSPlugin=mcs/user
MCSParameters=enforced,select,privatedata
PrivateData=jobs,nodes
```

每個使用者：
- 只能看到自己的作業
- 只能看到正在執行自己作業的節點
- 節點不會與其他使用者共享

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| mcs/account 未啟用計費 | 需要設定 accounting |
| privatedata 未搭配 PrivateData | 兩者需同時設定 |
| mcs/label 無管理機制 | 使用 job_submit 外掛程式 |
| 忽略搶佔限制 | 了解 MCS 對搶佔的影響 |

### 建議

1. **選擇合適的外掛程式**：
   - 大多數情況 mcs/account 或 mcs/group 足夠
   - mcs/label 需要額外管理

2. **測試隱私設定**：
   ```bash
   # 以不同使用者身份測試
   su - testuser -c "squeue"
   su - testuser -c "scontrol show nodes"
   ```

3. **監控標籤使用**：
   ```bash
   squeue -O jobid,username,mcslabel | sort -k3
   ```

---

## 快速參考

### slurm.conf 設定

```
# MCS 外掛程式選擇
MCSPlugin=mcs/[none|account|group|user|label]

# MCS 參數
MCSParameters=[ondemand|enforced],[noselect|select|ondemandselect][,privatedata][:groups]

# 搭配隱私設定
PrivateData=jobs,nodes
```

### 命令選項

| 選項 | 說明 |
|------|------|
| `--mcs-label=X` | 設定作業的 MCS 標籤 |
| `--exclusive=mcs` | 根據 MCS 標籤請求節點獨佔 |

### 常用命令

| 命令 | 功能 |
|------|------|
| `squeue -O mcslabel` | 顯示作業 MCS 標籤 |
| `scontrol show job` | 顯示作業詳細資訊 |
| `scontrol show nodes` | 顯示節點 MCS 標籤 |
| `scontrol show config \| grep MCS` | 顯示 MCS 設定 |
| `scontrol update job X MCSLabel=Y` | 更新作業標籤 |

### 相關文件

- [搶佔](preempt.md) - 作業搶佔設定
- [共享可消耗資源](cons_tres_share.md) - 資源共享設定
- [計費](accounting.md) - 計費系統
