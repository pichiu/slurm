# Slurm 聯邦排程指南 (Federated Scheduling Guide)

## TL;DR

Slurm 聯邦 (Federation) 允許多個叢集以點對點方式協調排程作業。作業提交到本地叢集後會複製到聯邦中的其他叢集，每個叢集獨立嘗試排程該作業。聯邦作業具有跨所有叢集唯一的作業 ID。適用於需要跨叢集負載平衡和資源共享的環境。

---

## 翻譯

### 概觀

Slurm 包含建立叢集聯邦並在叢集之間以點對點方式排程作業的支援。提交到聯邦的作業會獲得一個在聯邦中所有叢集間唯一的作業 ID。作業會提交到本地叢集（slurm.conf 中定義的叢集），然後複製到聯邦中的各個叢集。每個叢集隨後根據自己的排程策略獨立嘗試排程該作業。叢集會與「起源叢集」（作業提交的叢集）協調來排程作業。

**注意**：這不適用於高吞吐量環境。如果每天排程超過 50,000 個作業，請考慮設定較少的叢集來提交兄弟作業，或僅將負載導向本地叢集（例如可使用 `--cluster-constraint=` 或 `-M` 提交選項）。

---

### 設定

#### 建立聯邦

使用 sacctmgr 命令在資料庫中建立聯邦並將叢集加入聯邦：

```bash
# 建立聯邦
sacctmgr add federation <federation_name> [clusters=<list_of_clusters>]

# 範例
sacctmgr add federation myfed clusters=cluster1,cluster2,cluster3
```

#### 管理聯邦成員

叢集可以從聯邦新增或移除：

**注意**：一個叢集一次只能是一個聯邦的成員。

```bash
# 新增叢集到聯邦
sacctmgr modify federation myfed set clusters+=cluster4

# 從聯邦移除叢集
sacctmgr modify federation myfed set clusters-=cluster4

# 設定叢集的聯邦
sacctmgr modify cluster cluster1 set federation=myfed

# 從叢集移除聯邦關聯
sacctmgr modify cluster cluster1 set federation=
```

**注意**：如果在未先排空 (drain) 的情況下從聯邦移除叢集，已移除叢集上執行中的作業將繼續作為非聯邦作業執行。

#### 刪除聯邦

```bash
sacctmgr delete federation <federation_name>
```

#### 設定叢集特性

可以為叢集指定通用特性，並在提交時使用 `--cluster-constraint` 選項請求：

```bash
sacctmgr modify cluster cluster1 set features+=highmem,gpu
sacctmgr modify cluster cluster1 set features-=highmem
```

#### 設定叢集聯邦狀態

```bash
sacctmgr modify cluster <cluster_name> set fedstate=<state>
```

可用狀態：

| 狀態 | 說明 |
|------|------|
| **ACTIVE** | 叢集主動接受和排程聯邦作業 |
| **INACTIVE** | 叢集不會排程或接受任何作業 |
| **DRAIN** | 叢集不接受新作業，但讓現有聯邦作業完成 |
| **DRAIN+REMOVE** | 叢集不接受新作業，所有聯邦作業完成後自動從聯邦移除 |

#### 查看聯邦設定

```bash
# 查看聯邦設定
sacctmgr show federation [tree]
sacctmgr show cluster withfed

# 從控制器查看聯邦狀態
scontrol show federation
```

#### 預設聯邦檢視

預設情況下，狀態命令顯示本地檢視。可透過以下設定啟用預設聯邦檢視：

```
FederationParameters=fed_display
```

---

### 聯邦作業 ID

提交到聯邦的作業會獲得一個聯邦作業 ID。聯邦中的作業 ID 在所有叢集間是唯一的。聯邦作業 ID 使用無符號 32 位元整數來分配叢集 ID 和叢集本地 ID：

```
位元 0-25:  本地作業 ID
位元 26-31: 叢集起源 ID
```

聯邦作業 ID 允許控制器透過查看作業的叢集起源 ID 來得知作業是從哪個叢集提交的。

---

### 作業提交

當聯邦叢集收到作業提交時，它會將作業副本（**兄弟作業**）提交到每個符合條件的叢集。然後每個叢集獨立嘗試排程該作業。

#### 指定目標叢集

使用 `-M,--clusters` 選項：

```bash
# 提交到 cluster2 或 cluster3
cluster1$ sbatch -Mcluster2,cluster3 script.sh
```

作業將提交到 cluster2 或 cluster3，且只會在 cluster2 和 cluster3 建立兄弟作業，即使聯邦中有更多叢集。

#### 使用叢集特性約束

使用 `--cluster-constraint` 選項：

```bash
# 只提交到有 highmem 特性的叢集
sbatch --cluster-constraint=highmem script.sh

# 排除有 highmem 特性的叢集（注意引號）
sbatch --cluster-constraint='!highmem' script.sh
```

**注意**：使用 `!` 選項時，請加上引號以防止 shell 解釋 `!`。

同時使用兩個選項時，起源叢集只會將兄弟作業提交到同時滿足兩個條件的叢集。

#### 保留和相依作業

保留 (held) 或相依 (dependent) 作業會保留在起源叢集上，直到它們被釋放或不再相依，此時才會提交到聯邦中其他可行的叢集。

---

### 作業排程

聯邦中的每個叢集獨立嘗試排程每個作業，但需要與**起源叢集**協調以分配資源給聯邦作業。

排程流程：
1. 叢集決定可以嘗試為作業分配資源
2. 與起源叢集通訊，確認沒有其他叢集正在嘗試分配資源
3. 如果沒有衝突，嘗試分配資源
4. 成功則通知起源叢集，起源叢集通知其他叢集移除兄弟作業（進入**撤銷** revoked 狀態）
5. 失敗則通知起源叢集，讓其他叢集嘗試排程

如果起源叢集停機，遠端兄弟會與作業的可行兄弟協調來排程作業。當起源叢集恢復時，會與其他兄弟同步。

---

### 作業重新排入佇列

當聯邦作業重新排入佇列時，起源叢集會被通知，然後將新的兄弟作業提交到可行叢集，聯邦作業有資格在與之前執行的叢集不同的叢集上啟動。

slurm.conf 選項 `RequeueExit` 和 `RequeueExitHold` 由起源叢集控制。

---

### 互動式作業

互動式作業（使用 srun 和 salloc 提交的作業）可以提交到本地叢集並從不同叢集獲得分配。當 salloc 作業分配由非本地叢集授予時，會設定新的環境變數 `SLURM_WORKING_CLUSTER`，包含遠端兄弟叢集的 IP 位址、通訊埠和 RPC 版本，以便任何 srun 知道要與哪個叢集通訊。

**注意**：
- 所有計算節點必須對所有提交主機可存取
- Slurm 中 MPI 介面的目前實作要求執行 srun 的主機上的 SlurmdSpooldir 與分配中計算節點上的相同

---

### 取消作業

聯邦中的取消請求會取消執行中的兄弟作業或所有等待中的兄弟作業。

```bash
# 取消作業
scancel 12345

# 只移除特定兄弟作業
scancel --sibling=cluster2 12345
```

---

### 作業修改

作業修改會路由到起源叢集，起源叢集會將變更推送到每個兄弟作業。

---

### 作業陣列

目前，作業陣列只在起源叢集上執行。

---

### 狀態命令

預設情況下，狀態命令（squeue、sinfo、sprio、sacct、sreport）顯示本地叢集的檢視。

#### 啟用聯邦檢視

```bash
# 使用 --federation 選項
squeue --federation
sinfo --federation

# 或在 slurm.conf 設定預設聯邦檢視
FederationParameters=fed_display
```

可使用 `--local` 選項覆蓋聯邦檢視。使用 `--clusters,-M` 選項也會覆蓋聯邦檢視。

#### squeue 特殊選項

```bash
# 顯示每個兄弟作業而非合併
squeue --sibling
```

新的長格式選項：
- `cluster`：執行作業的叢集名稱
- `siblingsactive`：存在聯邦兄弟作業的叢集名稱
- `siblingsviable`：可執行聯邦兄弟作業的叢集名稱

```bash
# 按叢集排序
squeue -S cluster
```

#### sinfo

在聯邦檢視中，sinfo 會在一個檢視中顯示每個叢集的分割區，叢集名稱會與每個分割區一起顯示。

```bash
# 使用 %V 格式選項顯示叢集名稱
sinfo --federation -o "%V %P %a %D"

# 按叢集排序
sinfo --federation -S %V
```

#### sacct

預設情況下，sacct 不會顯示「撤銷」作業，只顯示執行作業的叢集資訊。

```bash
# 顯示撤銷的作業
sacct --duplicate
```

#### scontrol

支援聯邦檢視的選項：
- `show [--federation|--sibling] jobs`
- `show [--federation] steps`
- `completing`

在聯邦中處理的選項（會路由到起源叢集）：
- `hold`、`uhold`、`release`
- `requeue`、`requeuehold`
- `suspend`
- `update job`

---

## 說明

### 術語對照

| 術語 | 說明 |
|------|------|
| **聯邦作業 (Federated Job)** | 提交到聯邦叢集的作業，具有跨所有叢集唯一的作業 ID |
| **兄弟作業 (Sibling Job)** | 提交到其他聯邦叢集的聯邦作業副本 |
| **本地叢集 (Local Cluster)** | slurm.conf 中定義的叢集，命令預設與之通訊 |
| **起源叢集 (Origin Cluster)** | 聯邦作業最初提交的叢集，負責協調通訊 |
| **兄弟叢集 (Sibling Cluster)** | 與兄弟作業相關聯的叢集 |
| **撤銷狀態 (Revoked State, RV)** | 當另一個兄弟被分配節點時，兄弟作業進入的狀態 |
| **可行兄弟 (Viable Sibling)** | 根據請求的叢集、叢集特性和叢集狀態，有資格執行兄弟作業的叢集 |
| **活躍兄弟 (Active Sibling)** | 有兄弟作業且能夠排程該作業的叢集 |

### 聯邦 vs 多叢集操作

| 功能 | 聯邦排程 | 多叢集操作 |
|------|----------|------------|
| 作業複製 | 自動複製到多個叢集 | 只提交到一個叢集 |
| 作業 ID | 跨叢集唯一 | 每個叢集獨立 |
| 排程協調 | 叢集間協調 | 無協調 |
| 適用場景 | 負載平衡、高可用性 | 簡單的多叢集環境 |
| 複雜度 | 較高 | 較低 |

---

## 實務範例

### 建立和設定聯邦

```bash
# 建立包含三個叢集的聯邦
sacctmgr add federation production_fed clusters=cluster1,cluster2,cluster3

# 設定叢集特性
sacctmgr modify cluster cluster1 set features=highmem,ssd
sacctmgr modify cluster cluster2 set features=gpu,nvme
sacctmgr modify cluster cluster3 set features=standard

# 查看聯邦設定
sacctmgr show federation tree
```

### 提交聯邦作業

```bash
# 提交到任何可用叢集
sbatch job.sh

# 只提交到有 GPU 的叢集
sbatch --cluster-constraint=gpu job.sh

# 只提交到特定叢集
sbatch -M cluster1,cluster2 job.sh

# 排除某些叢集
sbatch --cluster-constraint='!highmem' job.sh
```

### 監控聯邦作業

```bash
# 查看聯邦中的所有作業
squeue --federation

# 查看特定作業的兄弟作業
squeue --sibling -j 12345

# 查看詳細的聯邦資訊
squeue --federation -o "%.10i %.9P %.8T %.10M %V %S"

# 查看聯邦狀態
scontrol show federation
```

### 管理聯邦叢集

```bash
# 將叢集設為維護模式（排空）
sacctmgr modify cluster cluster1 set fedstate=drain

# 從聯邦移除叢集（完成現有作業後）
sacctmgr modify cluster cluster1 set fedstate=drain+remove

# 重新啟用叢集
sacctmgr modify cluster cluster1 set fedstate=active
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 在高吞吐量環境使用聯邦 | 每天超過 50,000 作業時考慮減少聯邦叢集數量 |
| 未考慮網路延遲 | 確保叢集間網路連接穩定 |
| 作業陣列預期跨叢集執行 | 作業陣列目前只在起源叢集執行 |
| 互動式作業計算節點不可達 | 確保所有計算節點對所有提交主機可存取 |

### 限制

1. 因資源（分割區、節點數等）在本地叢集失敗的聯邦作業會被拒絕，不會提交到其他兄弟叢集
2. 作業陣列只在提交的叢集上執行
3. 作業修改必須在起源叢集成功才會推送到遠端叢集的兄弟作業
4. sview 中除作業外的修改在聯邦檢視中被停用
5. 聯邦檢視中 sview 網格被停用

### 最佳實務

1. **設定叢集特性**：為每個叢集設定有意義的特性，方便使用者選擇
2. **監控聯邦狀態**：定期檢查 `scontrol show federation`
3. **計畫維護**：使用 DRAIN 狀態進行維護
4. **日誌監控**：監控起源叢集的日誌以追蹤聯邦活動

---

## 快速參考

### sacctmgr 聯邦命令

| 命令 | 功能 |
|------|------|
| `sacctmgr add federation <name>` | 建立聯邦 |
| `sacctmgr delete federation <name>` | 刪除聯邦 |
| `sacctmgr modify federation <name> set clusters+=<list>` | 新增叢集 |
| `sacctmgr modify federation <name> set clusters-=<list>` | 移除叢集 |
| `sacctmgr modify cluster <name> set fedstate=<state>` | 設定狀態 |
| `sacctmgr show federation` | 查看聯邦 |

### 聯邦狀態

| 狀態 | 說明 |
|------|------|
| ACTIVE | 主動接受和排程作業 |
| INACTIVE | 不排程或接受作業 |
| DRAIN | 不接受新作業，完成現有作業 |
| DRAIN+REMOVE | 排空後自動從聯邦移除 |

### 提交選項

| 選項 | 說明 |
|------|------|
| `-M,--clusters=<list>` | 指定目標叢集 |
| `--cluster-constraint=<features>` | 要求叢集特性 |
| `--cluster-constraint='!<features>'` | 排除叢集特性 |

### 狀態命令選項

| 選項 | 說明 |
|------|------|
| `--federation` | 顯示聯邦檢視 |
| `--local` | 顯示本地檢視 |
| `--sibling` | 顯示每個兄弟作業 |

### slurm.conf 設定

```
# 預設顯示聯邦檢視
FederationParameters=fed_display
```
