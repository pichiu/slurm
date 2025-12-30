# Slurm Prolog 和 Epilog 指南 (Prolog and Epilog Guide)

## TL;DR

Slurm 支援多種 prolog 和 epilog 程式，用於在作業執行前後自動執行腳本。Prolog 在作業開始前執行，Epilog 在作業結束後執行。這些腳本可用於環境設定、資源清理、使用者驗證等目的。腳本應簡短，不應呼叫 Slurm 命令。

---

## 翻譯

### 概觀

Slurm 支援多種 prolog 和 epilog 程式。基於安全原因，這些程式沒有設定搜尋路徑。請在程式中指定完整路徑名稱或設定 `PATH` 環境變數。

---

### 作業分配的 Prolog/Epilog

以下表格說明作業分配可用的 prolog 和 epilog，以及它們何時何地執行：

| 參數 | 位置 | 呼叫者 | 使用者 | 執行時機 |
|------|------|--------|--------|----------|
| **Prolog** (slurm.conf) | 計算節點 | slurmd 守護程式 | SlurmdUser (通常是 root) | 該節點上第一個作業或作業步驟啟動時（預設）；`PrologFlags=Alloc` 會強制在作業分配時執行 |
| **PrologSlurmctld** (slurm.conf) | 頭節點 (slurmctld 執行處) | slurmctld 守護程式 | SlurmctldUser | 作業分配時 |
| **Epilog** (slurm.conf) | 計算節點 | slurmd 守護程式 | SlurmdUser (通常是 root) | 作業終止時 |
| **EpilogSlurmctld** (slurm.conf) | 頭節點 (slurmctld 執行處) | slurmctld 守護程式 | SlurmctldUser | 作業終止時 |

---

### 作業步驟的 Prolog/Epilog

以下表格說明作業步驟分配可用的 prolog 和 epilog：

| 參數 | 位置 | 呼叫者 | 使用者 | 執行時機 |
|------|------|--------|--------|----------|
| **SrunProlog** (slurm.conf) 或 `srun --prolog` | srun 呼叫節點 | srun 命令 | 呼叫 srun 的使用者 | 啟動作業步驟之前 |
| **TaskProlog** (slurm.conf) | 計算節點 | slurmstepd 守護程式 | 呼叫 srun 的使用者 | 啟動作業步驟之前 |
| `srun --task-prolog` | 計算節點 | slurmstepd 守護程式 | 呼叫 srun 的使用者 | 啟動作業步驟之前 |
| **TaskEpilog** (slurm.conf) | 計算節點 | slurmstepd 守護程式 | 呼叫 srun 的使用者 | 作業步驟完成時 |
| `srun --task-epilog` | 計算節點 | slurmstepd 守護程式 | 呼叫 srun 的使用者 | 作業步驟完成時 |
| **SrunEpilog** (slurm.conf) 或 `srun --epilog` | srun 呼叫節點 | srun 命令 | 呼叫 srun 的使用者 | 作業步驟完成時 |

---

### Prolog 執行時機

預設情況下，Prolog 腳本只在節點首次看到新分配的作業步驟時執行；它不會在分配授予時立即執行 Prolog。如果某個分配的作業步驟沒有在某節點上執行，該節點永遠不會為該分配執行 Prolog。此行為可透過 `PrologFlags` 參數變更。

另一方面，Epilog 總是在分配釋放時在分配的每個節點上執行。

---

### 多重腳本執行順序

如果指定多個 prolog 和/或 epilog 腳本（例如 `/etc/slurm/prolog.d/*`），它們將按反向字母順序執行（z-a -> Z-A -> 9-0）。

---

### 腳本設計建議

Prolog 和 Epilog 腳本應設計得盡可能簡短，且不應呼叫 Slurm 命令（如 squeue、scontrol、sacctmgr 等）。長時間執行的腳本可能導致作業啟動或結束時的排程問題。這些腳本中的 Slurm 命令可能導致效能問題，不應使用。

---

### TaskProlog 特殊功能

任務 prolog 以與要啟動的使用者任務相同的環境執行。該程式的標準輸出會被讀取並處理如下：

| 命令 | 效果 |
|------|------|
| `export name=value` | 為使用者任務設定環境變數 |
| `unset name` | 從使用者任務清除環境變數 |
| `print ...` | 寫入任務的標準輸出 |

**TaskProlog 範例腳本**：

```bash
#!/bin/bash

# TaskProlog 腳本可用於執行作業步驟前的任何準備工作，
# 也可用於修改使用者的環境。有兩個主要機制，
# 依賴於將命令印出到 stdout：

# 為使用者設定變數
echo "export VARIABLE_1=HelloWorld"

# 為使用者取消設定變數
echo "unset MANPATH"

# 如果需要也可以印出訊息
echo "print 此訊息已透過 TaskProlog 印出"
```

上述功能僅限於任務 prolog 腳本。

---

### 失敗處理

| 腳本類型 | 失敗時的行為 |
|----------|--------------|
| **Epilog** | 節點會被設為 DRAIN 狀態 |
| **EpilogSlurmctld** | 僅記錄日誌 |
| **Prolog** | 節點會被設為 DRAIN 狀態，作業會重新排入佇列並保留（除非設定 `nohold_on_prolog_fail`） |
| **PrologSlurmctld** | 作業會重新排入佇列。僅批次作業可重新排入佇列；互動式作業（salloc 和 srun）會被取消 |
| **TaskProlog** | 任務會被取消 |
| **TaskEpilog** | 僅記錄日誌 |
| **SrunProlog** | 步驟會被取消 |
| **SrunEpilog** | 僅記錄日誌 |

---

### 環境變數

除非另有說明，以下環境變數對所有程式都可用：

#### GPU 相關

| 變數 | 說明 | 可用範圍 |
|------|------|----------|
| `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` | 應分配給作業的 GPU 百分比 | Prolog, Epilog |
| `CUDA_VISIBLE_DEVICES` | 作業分配的 GPU 裝置 | Prolog, Epilog |
| `GPU_DEVICE_ORDINAL` | GPU 裝置（同 CUDA_VISIBLE_DEVICES） | Prolog, Epilog |
| `ROCR_VISIBLE_DEVICES` | AMD GPU 裝置 | Prolog, Epilog |

#### 作業陣列相關

| 變數 | 說明 | 可用範圍 |
|------|------|----------|
| `SLURM_ARRAY_JOB_ID` | 作業陣列的作業 ID | PrologSlurmctld, SrunProlog, TaskProlog, EpilogSlurmctld, SrunEpilog, TaskEpilog |
| `SLURM_ARRAY_TASK_COUNT` | 陣列中的任務數量 | 同上 |
| `SLURM_ARRAY_TASK_ID` | 任務 ID | 同上 |
| `SLURM_ARRAY_TASK_MAX` | 最大任務 ID | 同上 |
| `SLURM_ARRAY_TASK_MIN` | 最小任務 ID | 同上 |
| `SLURM_ARRAY_TASK_STEP` | 任務 ID 步長 | 同上 |

#### 作業資訊

| 變數 | 說明 | 可用範圍 |
|------|------|----------|
| `SLURM_CLUSTER_NAME` | 執行作業的叢集名稱 | Prolog, PrologSlurmctld, Epilog, EpilogSlurmctld |
| `SLURM_CONF` | slurm.conf 位置 | Prolog, SrunProlog, TaskProlog, Epilog, SrunEpilog, TaskEpilog |
| `SLURM_JOB_ACCOUNT` | 作業使用的帳戶名稱 | 所有 |
| `SLURM_JOB_ID` / `SLURM_JOBID` | 作業 ID | 所有 |
| `SLURM_JOB_NAME` | 作業名稱 | PrologSlurmctld, SrunProlog, TaskProlog, EpilogSlurmctld, SrunEpilog, TaskEpilog |
| `SLURM_JOB_NODELIST` | 分配給作業的節點（hostlist 格式） | 所有 |
| `SLURM_JOB_NUM_NODES` | 分配給作業的節點數 | 所有 |
| `SLURM_JOB_PARTITION` | 作業執行的分割區 | 所有 |
| `SLURM_JOB_QOS` | 分配給作業的 QOS | 所有 |
| `SLURM_JOB_UID` | 作業擁有者的使用者 ID | 所有 |
| `SLURM_JOB_USER` | 作業擁有者的使用者名稱 | 所有 |
| `SLURM_JOB_GID` | 作業擁有者的群組 ID | 所有 |

#### 作業結束資訊（僅 Epilog）

| 變數 | 說明 |
|------|------|
| `SLURM_JOB_DERIVED_EC` | 所有作業步驟的最高結束代碼 |
| `SLURM_JOB_EXIT_CODE` | 作業腳本的結束代碼（wait() 格式） |
| `SLURM_JOB_EXIT_CODE2` | 作業腳本的結束代碼（格式：`<exit>:<sig>`） |

#### 任務相關（srun 環境）

| 變數 | 說明 |
|------|------|
| `SLURM_CPUS_ON_NODE` | 當前節點上作業可用的處理器數 |
| `SLURM_LOCALID` | 節點內任務的本地 ID |
| `SLURM_NODEID` | 多節點作業中當前節點的相對 ID |
| `SLURM_NTASKS` | 作業請求的任務數 |
| `SLURM_PROCID` | MPI rank（或相對程序 ID） |
| `SLURM_TASK_PID` | 任務啟動的程序 ID |
| `SLURMD_NODENAME` | 執行任務的節點名稱 |

#### 腳本識別

| 變數值 | 說明 |
|--------|------|
| `prolog_slurmctld` | PrologSlurmctld 正在執行 |
| `epilog_slurmctld` | EpilogSlurmctld 正在執行 |
| `prolog_slurmd` | Prolog 正在執行 |
| `epilog_slurmd` | Epilog 正在執行 |
| `prolog_task` | TaskProlog 正在執行 |
| `epilog_task` | TaskEpilog 正在執行 |
| `prolog_srun` | SrunProlog 正在執行 |
| `epilog_srun` | SrunEpilog 正在執行 |

---

## 說明

### Prolog vs Epilog 執行流程

```
作業分配
    ↓
PrologSlurmctld（控制節點）
    ↓
Prolog（計算節點，可選延遲到首次步驟）
    ↓
[作業步驟迴圈]
    SrunProlog → TaskProlog → 任務執行 → TaskEpilog → SrunEpilog
    ↓
Epilog（計算節點）
    ↓
EpilogSlurmctld（控制節點）
    ↓
作業完成
```

### 執行身份差異

| 腳本 | 執行身份 | 原因 |
|------|----------|------|
| Prolog/Epilog | root (SlurmdUser) | 需要系統層級操作 |
| PrologSlurmctld/EpilogSlurmctld | SlurmctldUser | 控制器權限 |
| TaskProlog/TaskEpilog | 作業使用者 | 使用者環境操作 |
| SrunProlog/SrunEpilog | 作業使用者 | 使用者層級操作 |

---

## 實務範例

### 基本 Prolog 腳本

```bash
#!/bin/bash
# /etc/slurm/prolog
# 作業開始前執行的準備工作

# 記錄作業開始
logger "Slurm Job ${SLURM_JOB_ID} starting on $(hostname)"

# 清理暫存目錄
rm -rf /tmp/slurm-${SLURM_JOB_ID} 2>/dev/null
mkdir -p /tmp/slurm-${SLURM_JOB_ID}
chown ${SLURM_JOB_UID}:${SLURM_JOB_GID} /tmp/slurm-${SLURM_JOB_ID}

# 設定 GPU 環境（如果有 GPU）
if [ -n "${CUDA_VISIBLE_DEVICES}" ]; then
    logger "Job ${SLURM_JOB_ID} allocated GPUs: ${CUDA_VISIBLE_DEVICES}"
fi

exit 0
```

### 基本 Epilog 腳本

```bash
#!/bin/bash
# /etc/slurm/epilog
# 作業結束後執行的清理工作

# 記錄作業結束
logger "Slurm Job ${SLURM_JOB_ID} finished on $(hostname) with exit code ${SLURM_JOB_EXIT_CODE}"

# 清理作業暫存目錄
rm -rf /tmp/slurm-${SLURM_JOB_ID}

# 終止任何殘留程序
pkill -9 -u ${SLURM_JOB_UID} 2>/dev/null

# 清理共享記憶體段
ipcrm -M $(ipcs -m | grep ${SLURM_JOB_UID} | awk '{print $1}') 2>/dev/null

exit 0
```

### TaskProlog 設定使用者環境

```bash
#!/bin/bash
# /etc/slurm/task_prolog
# 為每個任務設定環境

# 設定作業專屬暫存目錄
echo "export TMPDIR=/tmp/slurm-${SLURM_JOB_ID}"
echo "export SCRATCH=/scratch/slurm-${SLURM_JOB_ID}"

# 載入模組（透過 print 顯示訊息）
echo "print 正在載入預設模組..."

# 設定 GPU 相關環境
if [ -n "${SLURM_JOB_GPUS}" ]; then
    echo "export CUDA_HOME=/usr/local/cuda"
    echo "export PATH=\${CUDA_HOME}/bin:\${PATH}"
fi
```

### PrologSlurmctld 驗證使用者配額

```bash
#!/bin/bash
# /etc/slurm/prolog_slurmctld
# 在控制節點驗證作業

# 檢查使用者配額
USER_QUOTA=$(sacctmgr -n -P show assoc user=${SLURM_JOB_USER} format=grptres)
# 注意：實際環境中應避免在 prolog 中呼叫 Slurm 命令
# 這只是示範概念

# 記錄作業分配
echo "$(date): Job ${SLURM_JOB_ID} allocated to user ${SLURM_JOB_USER}" >> /var/log/slurm/job_allocations.log

exit 0
```

### slurm.conf 設定範例

```
# Prolog 和 Epilog 設定
Prolog=/etc/slurm/prolog
Epilog=/etc/slurm/epilog
PrologSlurmctld=/etc/slurm/prolog_slurmctld
EpilogSlurmctld=/etc/slurm/epilog_slurmctld

# 任務層級腳本
TaskProlog=/etc/slurm/task_prolog
TaskEpilog=/etc/slurm/task_epilog

# Srun 層級腳本
SrunProlog=/etc/slurm/srun_prolog
SrunEpilog=/etc/slurm/srun_epilog

# Prolog 旗標設定
PrologFlags=Alloc,Serial

# 排程器參數（prolog 失敗時不保留作業）
SchedulerParameters=nohold_on_prolog_fail
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 在 Prolog/Epilog 中呼叫 squeue/scontrol | 避免呼叫 Slurm 命令，可能導致死結 |
| 腳本執行時間過長 | 保持腳本簡短，複雜操作應非同步執行 |
| 忘記設定執行權限 | `chmod +x /etc/slurm/prolog` |
| 使用相對路徑 | 始終使用絕對路徑 |
| Epilog 失敗未處理 | 確保 Epilog 腳本健壯，適當處理錯誤 |
| 未考慮並行執行 | 多個作業可能同時執行腳本，避免競爭條件 |

### 除錯技巧

```bash
# 測試 Prolog 腳本（手動執行）
export SLURM_JOB_ID=12345
export SLURM_JOB_UID=$(id -u testuser)
export SLURM_JOB_GID=$(id -g testuser)
/etc/slurm/prolog

# 查看 Prolog/Epilog 日誌
grep -i "prolog\|epilog" /var/log/slurm/slurmd.log

# 檢查節點為何處於 DRAIN 狀態
scontrol show node node01 | grep Reason
```

### 效能建議

1. **保持腳本簡短**：目標執行時間 < 5 秒
2. **避免網路操作**：NFS 操作可能導致超時
3. **非同步處理**：複雜操作可背景執行
4. **錯誤處理**：始終返回適當的退出碼

---

## 快速參考

### 腳本位置和執行者

| 腳本 | 設定參數 | 執行位置 | 執行身份 |
|------|----------|----------|----------|
| Prolog | `Prolog` | 計算節點 | root |
| Epilog | `Epilog` | 計算節點 | root |
| PrologSlurmctld | `PrologSlurmctld` | 控制節點 | SlurmctldUser |
| EpilogSlurmctld | `EpilogSlurmctld` | 控制節點 | SlurmctldUser |
| TaskProlog | `TaskProlog` | 計算節點 | 作業使用者 |
| TaskEpilog | `TaskEpilog` | 計算節點 | 作業使用者 |
| SrunProlog | `SrunProlog` | 提交節點 | 作業使用者 |
| SrunEpilog | `SrunEpilog` | 提交節點 | 作業使用者 |

### PrologFlags 選項

| 旗標 | 說明 |
|------|------|
| `Alloc` | 在作業分配時立即執行 Prolog，而非等到首次步驟 |
| `Serial` | 在節點上序列執行 Prolog（一次一個） |
| `X11` | 設定 X11 轉發 |
| `Contain` | 等待 Prolog 完成後才允許使用者存取節點 |

### 常用環境變數速查

```bash
# 作業識別
$SLURM_JOB_ID      # 作業 ID
$SLURM_JOB_NAME    # 作業名稱
$SLURM_JOB_USER    # 使用者名稱
$SLURM_JOB_UID     # 使用者 ID

# 資源資訊
$SLURM_JOB_NODELIST      # 節點列表
$SLURM_JOB_NUM_NODES     # 節點數
$SLURM_JOB_CPUS_PER_NODE # 每節點 CPU 數

# 結束資訊（僅 Epilog）
$SLURM_JOB_EXIT_CODE     # 結束代碼
$SLURM_JOB_EXIT_CODE2    # 結束代碼（含信號）
```

### 相關文件

- [SPANK](spank.html) - 另一種在各點執行邏輯的機制
- [slurm.conf](slurm.conf.html) - 主要設定檔
- [故障排除](troubleshoot.md) - 問題診斷
