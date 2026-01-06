# Slurm 節能指南 (Power Saving Guide)

## TL;DR

Slurm 提供節點掛起（關機/節能模式）和恢復（開機）的整合機制。閒置 SuspendTime 的節點會被 SuspendProgram 掛起，需要時由 ResumeProgram 恢復。此功能可用於傳統節能或雲端彈性運算（AWS、GCP、Azure）。關鍵設定：ResumeProgram、SuspendProgram、SuspendTime、ResumeTimeout。

---

## 翻譯

### 概觀

Slurm 提供一個整合機制，讓節點可以按需或依請求被掛起（關機、進入節能模式）和恢復（開機、從節能模式恢復）。

- 保持閒置 **SuspendTime** 的節點會被 **SuspendProgram** 掛起
- 節點會在 **SuspendTimeout** 期間無法排程
- 節點會自動被 **ResumeProgram** 恢復以完成分配給它們的工作
- 未能在 **ResumeTimeout** 內註冊的節點會變為 DOWN 狀態，其分配的作業會重新排入佇列
- 可手動請求節點電源狀態：`scontrol update nodename=<nodename> state=power_<down|up>`
- 恢復和掛起節點的速率可透過 **ResumeRate** 和 **SuspendRate** 控制

Slurm 可設定透過 API 管理任何雲端供應商（如 Amazon Web Services、Google Cloud Platform、Microsoft Azure）的運算資源來達成節能。這些資源可與現有叢集結合以處理過量工作負載（雲端爆發）或作為獨立自給自足的叢集運作。

#### 啟用節能操作的必要設定

- **ResumeProgram** 和 **SuspendProgram** 必須定義，值必須是有效的程式路徑
- **ResumeTimeout** 和 **SuspendTimeout** 必須定義（全域或至少一個分割區）
- **SuspendTime** 必須定義（全域或至少一個分割區），且不能是 INFINITE 或 -1
- **ResumeRate** 和 **SuspendRate** 必須大於或等於 0

必須重新啟動 slurmctld 守護程式才能初次啟用節能操作。

---

### 設定

#### 主要設定參數

| 參數 | 說明 | 預設值 |
|------|------|--------|
| **ResumeProgram** | 從節能模式恢復節點的程式 | 無 |
| **SuspendProgram** | 將節點置於節能模式的程式 | 無 |
| **ResumeTimeout** | 節點恢復請求到可用的最大時間（秒） | 60 |
| **SuspendTimeout** | 節點掛起請求到關機完成的最大時間（秒） | 30 |
| **SuspendTime** | 節點閒置多久後可進入節能模式（秒） | -1（停用）|
| **ResumeRate** | 每分鐘最大恢復節點數 | 300 |
| **SuspendRate** | 每分鐘最大掛起節點數 | 60 |
| **ResumeFailProgram** | 節點恢復失敗時執行的程式 | 無 |

#### 排除設定

| 參數 | 說明 |
|------|------|
| **SuspendExcNodes** | 不受掛起/恢復邏輯影響的節點 |
| **SuspendExcParts** | 節點永不進入節能模式的分割區列表 |
| **SuspendExcStates** | 不自動關機的節點狀態（CLOUD、DOWN、DRAIN 等）|

#### 排程器參數

| 參數 | 說明 |
|------|------|
| **salloc_wait_nodes** | salloc 等待所有節點準備好才返回 |
| **sbatch_wait_nodes** | sbatch 腳本等待所有節點準備好才啟動 |

#### SlurmctldParameters

| 參數 | 說明 |
|------|------|
| **cloud_dns** | 雲端節點在 DNS 中，避免等待 IP 位址 |
| **idle_on_node_suspend** | 掛起節點時標記為閒置 |
| **node_reg_mem_percent** | 節點允許註冊的記憶體百分比（CLOUD 節點預設 90%）|
| **power_save_interval** | power_save 執行緒檢查頻率（預設 10 秒）|

---

### 節點設定

| 參數 | 說明 |
|------|------|
| **Feature** | 可與 --constraint 選項配合使用的節點特性 |
| **NodeName** | Slurm 參照節點的名稱（建議使用數字後綴）|
| **State** | 按需新增的節點應設為 CLOUD |
| **Weight** | 資源使用順序權重（較低的先使用，預設 1）|

---

### 分割區設定

| 參數 | 說明 |
|------|------|
| **PowerDownOnIdle** | 設為 YES 時，節點在分配作業後變為閒置時會關機 |
| **ResumeTimeout** | 分割區層級的恢復超時（覆蓋全域設定）|
| **SuspendTime** | 分割區層級的掛起時間（設為 INFINITE 停用此分割區的掛起）|
| **SuspendTimeout** | 分割區層級的掛起超時 |

---

### 節點生命週期

節能操作相關的節點狀態：

| 狀態 | 符號 | 說明 |
|------|------|------|
| POWER_DOWN | ! | 關機請求，當節點不再執行作業時執行 SuspendProgram |
| POWER_UP | | 開機請求，在可能時執行 ResumeProgram |
| POWERED_DOWN | ~ | 節點已關機或處於節能模式 |
| POWERING_DOWN | % | 節點正在關機過程中 |
| POWERING_UP | # | 節點正在開機過程中 |

---

### 手動電源管理

使用 scontrol 手動控制節點電源：

```bash
scontrol update nodename=<nodename> state=power_<down|down_asap|down_force|up>
```

| 狀態 | 說明 |
|------|------|
| **POWER_DOWN** | 明確將節點置於節能模式 |
| **POWER_DOWN_ASAP** | 排空節點並標記為關機（現有作業完成後）|
| **POWER_DOWN_FORCE** | 取消節點上所有作業，關機並重設為 IDLE |
| **POWER_UP** | 明確將節點從節能模式恢復 |
| **RESUME** | 將節點從 DRAIN/DOWN 改為 IDLE |

---

### 恢復和掛起程式

ResumeProgram 和 SuspendProgram 以 SlurmUser 身份在 slurmctld 執行的節點上執行。

**ResumeProgram 範例：**
```bash
#!/bin/bash
hosts=$(scontrol show hostnames "$1")
logfile=/var/log/power_save.log
echo "$(date) Resume invoked $0 $*" >>$logfile
for host in $hosts
do
    sudo node_startup "$host"
done
exit 0
```

**SuspendProgram 範例：**
```bash
#!/bin/bash
hosts=$(scontrol show hostnames "$1")
logfile=/var/log/power_save.log
echo "$(date) Suspend invoked $0 $*" >>$logfile
for host in $hosts
do
    sudo node_shutdown "$host"
done
exit 0
```

#### SLURM_RESUME_FILE

ResumeProgram 可讀取 SLURM_RESUME_FILE 環境變數指定的 JSON 檔案，獲取作業到節點的對應：

```json
{
  "all_nodes_resume" : "cloud[1-3,7-8]",
  "jobs" : [
    {
      "extra" : "An arbitrary string from --extra",
      "features" : "c1,c2",
      "job_id" : 140814,
      "nodes_alloc" : "cloud[1-4]",
      "nodes_resume" : "cloud[1-3]",
      "partition" : "cloud"
    }
  ]
}
```

---

### 容錯

- slurmctld 正常終止時，會等待最多 10 秒讓 SuspendProgram 或 ResumeProgram 終止
- slurmctld 關閉後，SLURM_RESUME_FILE 暫存檔不再可用
- ResumeProgram 應在啟動後 10 秒內使用 SLURM_RESUME_FILE

---

### 混合叢集

雲端節點可以放入自己的 Slurm 分割區，僅在使用者請求時使用或在工作負載超過可用資源時使用（雲端爆發）。

**範例設定：**
```
# slurm.conf
SelectType=select/cons_tres
SelectTypeParameters=CR_CORE_Memory

SuspendProgram=/usr/sbin/slurm_suspend
ResumeProgram=/usr/sbin/slurm_resume
SuspendTime=600
SuspendExcNodes=tux[0-127]
TreeWidth=128

NodeName=DEFAULT    Sockets=1 CoresPerSocket=4 ThreadsPerCore=2
NodeName=tux[0-127] Weight=1 Feature=local State=UNKNOWN
NodeName=ec[0-127]  Weight=8 Feature=cloud State=CLOUD

PartitionName=debug MaxTime=1:00:00 Nodes=tux[0-32] Default=YES
PartitionName=batch MaxTime=8:00:00 Nodes=tux[0-127],ec[0-127]
```

**避免排除本地節點的替代設定：**
```
# 使用分割區層級 SuspendTime
SuspendTime=INFINITE  # 全域設為無限
PartitionName=cloud Nodes=ec[0-127] SuspendTime=600  # 僅雲端分割區掛起
```

---

### 雲端計費

雲端實例資訊可儲存在資料庫中。

**slurmd 啟動時設定：**
```bash
slurmd --instance-id=12345 --instance-type=m7g.medium --extra="arbitrary string"
```

**使用 scontrol 設定：**
```bash
scontrol update nodename=n1 instanceid=12345 instancetype=m7g.medium extra="arbitrary string"
```

**查看資訊：**
```bash
# 控制器
scontrol show nodes n1 | grep "NodeName\|Extra\|Instance"

# 資料庫
sacctmgr show instance format=nodename,instanceid,instancetype,extra
```

---

## 說明

### 節能 vs 雲端爆發

| 用途 | 設定重點 |
|------|----------|
| **傳統節能** | 本地節點閒置時關機省電 |
| **雲端爆發** | 按需啟動雲端節點處理過量工作負載 |
| **純雲端叢集** | 所有節點都是 State=CLOUD |

### 節點權重與分配順序

```
1. 已開機節點（按權重）
2. 正在開機節點（按權重）
3. 已關機節點（按權重）
```

---

## 實務範例

### AWS 雲端爆發設定

```
# slurm.conf
SuspendProgram=/etc/slurm/scripts/aws_suspend.sh
ResumeProgram=/etc/slurm/scripts/aws_resume.sh
ResumeFailProgram=/etc/slurm/scripts/aws_resume_fail.sh
SuspendTime=300
ResumeTimeout=600
SuspendTimeout=120
ResumeRate=100
SuspendRate=100

NodeName=aws[1-100] State=CLOUD Feature=cloud CPUs=4 RealMemory=16000
PartitionName=cloud Nodes=aws[1-100] SuspendTime=300
```

**aws_resume.sh：**
```bash
#!/bin/bash
NODES=$1
LOGFILE=/var/log/slurm/power_save.log

echo "$(date) Resuming: $NODES" >> $LOGFILE

# 讀取 SLURM_RESUME_FILE 獲取作業資訊
if [ -f "$SLURM_RESUME_FILE" ]; then
    cat "$SLURM_RESUME_FILE" >> $LOGFILE
fi

for node in $(scontrol show hostnames "$NODES"); do
    # 啟動 AWS 實例
    aws ec2 start-instances --instance-ids $(get_instance_id $node)
done
```

### 監控節能狀態

```bash
# 查看節點電源狀態
sinfo -N -l | grep -E "(~|#|%|!)"

# 符號說明：
# ~ 已關機
# # 正在開機
# % 正在關機
# ! 待關機

# 查看詳細節點狀態
scontrol show node | grep -E "NodeName|State|Reason"
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| ResumeTimeout 過短 | 雲端節點需要較長時間開機，設定 300-600 秒 |
| 未設定 SuspendExcNodes | 排除本地節點避免意外關機 |
| 程式無執行權限 | 確保 SlurmUser 可執行程式 |
| 忽略 SLURM_RESUME_FILE | 在程式開始時讀取以獲取作業資訊 |
| 未記錄程式輸出 | 在腳本中加入日誌記錄 |

### 除錯技巧

```bash
# 啟用節能除錯日誌
# slurm.conf
DebugFlags=Power

# 查看節能相關日誌
grep -i "power\|suspend\|resume" /var/log/slurm/slurmctld.log

# 手動測試程式
/usr/sbin/slurm_resume "node[1-5]"
/usr/sbin/slurm_suspend "node[1-5]"
```

---

## 快速參考

### slurm.conf 設定範本

```
# 節能設定
SuspendProgram=/etc/slurm/suspend.sh
ResumeProgram=/etc/slurm/resume.sh
ResumeFailProgram=/etc/slurm/resume_fail.sh
SuspendTime=600
SuspendTimeout=120
ResumeTimeout=600
ResumeRate=300
SuspendRate=60

# 排除設定
SuspendExcNodes=local[0-99]
SuspendExcParts=debug
SuspendExcStates=DOWN,DRAIN,MAINTENANCE

# 雲端節點
NodeName=cloud[0-49] State=CLOUD Feature=cloud CPUs=4 RealMemory=16000
PartitionName=cloud Nodes=cloud[0-49] SuspendTime=300 PowerDownOnIdle=YES
```

### scontrol 命令

| 命令 | 功能 |
|------|------|
| `scontrol update nodename=X state=power_down` | 手動關機 |
| `scontrol update nodename=X state=power_up` | 手動開機 |
| `scontrol update nodename=X state=power_down_asap` | 排空後關機 |
| `scontrol update nodename=X state=power_down_force` | 強制關機 |
| `scontrol update SuspendExcNodes=X` | 更新排除節點 |

### 相關文件

- [動態節點](dynamic_nodes.md)
- [拓撲](topology.md)
- [聯邦排程](federation.md)
