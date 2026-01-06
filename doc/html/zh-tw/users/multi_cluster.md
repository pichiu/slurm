# Slurm 多叢集操作 (Multi-Cluster Operation)

## TL;DR

Slurm 支援多叢集操作，使用者可以透過 `-M, --clusters=` 選項將命令發送到多個叢集。當使用 sbatch、salloc 或 srun 指定叢集列表時，Slurm 會自動將作業提交到能提供最早開始時間的叢集。此功能需要配置 SlurmDBD 和正確的認證金鑰。

---

## 翻譯

### 多叢集操作

一個叢集（cluster）由單一 slurmctld 守護程式管理的所有節點組成。Slurm 提供將命令發送到其他叢集的能力，而不僅限於或額外包含執行命令的本地叢集。啟用此功能後，使用者可以向一個或多個叢集提交作業，並從這些遠端叢集接收狀態資訊。

**範例：**

```bash
juser@dawn> squeue -M dawn,dusk
CLUSTER: dawn
JOBID PARTITION   NAME   USER  ST   TIME NODES BP_LIST(REASON)
76897    pdebug  myJob  juser   R   4:10   128 dawn001[8-15]
76898    pdebug  myJob  juser   R   4:10   128 dawn001[16-23]
16899    pdebug  myJob  juser   R   4:10   128 dawn001[24-31]

CLUSTER: dusk
JOBID PARTITION   NAME   USER  ST   TIME NODES BP_LIST(REASON)
11950    pdebug   aJob  juser   R   4:20   128 dusk000[0-15]
11949    pdebug   aJob  juser   R   5:01   128 dusk000[48-63]
11946    pdebug   aJob  juser   R   6:35   128 dusk000[32-47]
11945    pdebug   aJob  juser   R   6:36   128 dusk000[16-31]
```

大多數 Slurm 用戶端命令都提供 `-M, --clusters=` 選項，可以與逗號分隔的叢集列表進行通訊。

當使用叢集列表呼叫 **sbatch**、**salloc** 或 **srun** 時，Slurm 會立即將作業提交到根據其等待和執行中作業佇列能提供最早開始時間的叢集。Slurm 不會在之後嘗試將作業遷移到（列表中）因執行中作業提前結束而釋放資源的不同叢集。

**注意**：為了讓 **salloc** 或 **srun** 在多叢集環境中使用 `-M, --clusters` 選項正常運作，計算節點必須能夠與提交主機互相存取。

---

### 多叢集設定

多叢集功能需要使用 SlurmDBD。slurm.conf 檔案中的 AccountingStorageType 必須設定為 accounting_storage/slurmdbd 外掛程式，且必須安裝 MUNGE 或認證金鑰以允許每個叢集與 SlurmDBD 通訊。請注意，如果需要，MUNGE 可以設定為在叢集內通訊和跨叢集通訊時使用不同的金鑰。詳情請參閱[計費](../administrators/accounting.md)文件。

設定完成後，指定 `-M, --clusters=` 選項的 Slurm 命令將對 `sacctmgr show clusters` 命令列出的所有叢集生效。

另請參閱 [Slurm 聯邦排程指南](federation.md)。

---

## 說明

### 叢集定義

| 術語 | 說明 |
|------|------|
| **叢集 (Cluster)** | 由單一 slurmctld 守護程式管理的所有節點 |
| **本地叢集** | 執行命令的叢集 |
| **遠端叢集** | 透過 `-M` 選項存取的其他叢集 |

### 作業路由行為

當使用 `-M cluster1,cluster2,cluster3` 提交作業時：

1. Slurm 查詢每個叢集的排程狀態
2. 計算每個叢集的預期開始時間
3. 將作業提交到能提供最早開始時間的叢集
4. 作業一旦提交，**不會**自動遷移到其他叢集

### 與聯邦 (Federation) 的區別

| 功能 | 多叢集操作 | 聯邦排程 |
|------|------------|----------|
| 作業路由 | 提交時一次性決定 | 可動態重新路由 |
| 設定複雜度 | 較簡單 | 較複雜 |
| 作業遷移 | 不支援 | 支援 |
| 適用場景 | 簡單的多叢集環境 | 需要負載平衡的環境 |

---

## 實務範例

### 查看多叢集狀態

```bash
# 查看多個叢集的作業佇列
squeue -M cluster1,cluster2

# 查看所有已設定叢集的作業佇列
squeue -M all

# 查看多個叢集的節點狀態
sinfo -M cluster1,cluster2

# 查看多個叢集的分割區資訊
sinfo -M cluster1,cluster2 -s
```

### 提交作業到多叢集

```bash
# 提交到能最快開始的叢集
sbatch -M cluster1,cluster2 my_script.sh

# 互動式作業到多叢集
salloc -M cluster1,cluster2 -N 4 -t 1:00:00

# 指定單一遠端叢集
sbatch -M remote_cluster my_script.sh
```

### 管理遠端叢集上的作業

```bash
# 取消遠端叢集上的作業
scancel -M remote_cluster 12345

# 查看遠端叢集上的作業詳情
scontrol -M remote_cluster show job 12345

# 修改遠端叢集上的作業
scontrol -M remote_cluster update job 12345 TimeLimit=2:00:00
```

### 查看叢集資訊

```bash
# 列出所有已設定的叢集
sacctmgr show clusters

# 查看叢集詳細資訊
sacctmgr show clusters format=cluster,controlhost,controlport,rpc

# 查看特定叢集的帳戶關聯
sacctmgr show assoc cluster=cluster1
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 SlurmDBD | 多叢集功能必須使用 SlurmDBD |
| 認證金鑰不匹配 | 確保所有叢集使用相同的 MUNGE 金鑰或正確設定跨叢集認證 |
| 計算節點無法存取 | salloc/srun 需要計算節點能與提交主機通訊 |
| 期望作業自動遷移 | 多叢集操作不會自動遷移作業，考慮使用聯邦排程 |

### 設定檢查清單

1. ✅ slurm.conf 中設定 `AccountingStorageType=accounting_storage/slurmdbd`
2. ✅ SlurmDBD 正常運行
3. ✅ 所有叢集已在 SlurmDBD 中註冊（`sacctmgr show clusters`）
4. ✅ MUNGE 金鑰正確設定
5. ✅ 網路防火牆允許叢集間通訊
6. ✅ DNS 或 /etc/hosts 正確設定叢集主機名稱

### 除錯技巧

```bash
# 檢查 SlurmDBD 連線
sacctmgr show clusters

# 測試遠端叢集連線
sinfo -M remote_cluster

# 查看詳細錯誤訊息
squeue -M remote_cluster -v

# 檢查認證
scontrol -M remote_cluster ping
```

---

## 快速參考

### 支援 -M 選項的命令

| 命令 | 功能 |
|------|------|
| `sbatch` | 提交批次作業 |
| `salloc` | 分配互動式資源 |
| `srun` | 執行作業步驟 |
| `squeue` | 查看作業佇列 |
| `sinfo` | 查看節點/分割區資訊 |
| `scancel` | 取消作業 |
| `scontrol` | 系統控制 |
| `sacct` | 作業計費資訊 |

### -M 選項語法

```bash
# 單一叢集
command -M cluster1

# 多個叢集（逗號分隔）
command -M cluster1,cluster2,cluster3

# 所有叢集
command -M all
```

### slurm.conf 關鍵設定

```
# 啟用 SlurmDBD 計費儲存
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=slurmdbd_hostname
```
