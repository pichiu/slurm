# Slurm 作業狀態代碼 (Job State Codes)

## TL;DR

Slurm 系統中每個作業都有一個狀態。狀態分為「基本狀態」(base states) 和「狀態旗標」(state flags)。使用 `squeue` 和 `sacct` 查看作業狀態時，會顯示單一狀態字串。常見狀態包括：PENDING（等待中）、RUNNING（執行中）、COMPLETED（完成）、FAILED（失敗）、CANCELLED（取消）等。

---

## 翻譯

### 作業狀態代碼

Slurm 系統中的每個作業都會被指派一個狀態。作業狀態的顯示方式取決於用來識別狀態的方法。

#### 概觀

在 Slurm 程式碼中，有**基本狀態**和**狀態旗標**兩種類型。每個作業都有一個基本狀態，並可能設置額外的狀態旗標。使用 [REST API](rest_quickstart.md) 時，會同時回傳基本狀態和目前的旗標。

當 [squeue](squeue.html) 和 [sacct](sacct.html) 指令報告作業狀態時，它們會以單一狀態來表示。兩者都能識別所有基本狀態，但並非所有狀態旗標都能識別。如果存在可識別的旗標，將會報告該旗標而非基本狀態。詳細資訊請參閱相關指令文件。

本頁列出程式碼中所有的作業代碼和旗標。提供的名稱是用於使用者介面輸出的字串表示。大多數情況下，程式碼中使用的名稱相同，只是在開頭加上 `JOB_`。若要更深入了解作業狀態和旗標，請在 [slurm.conf](slurm.conf.html) 中設置 `DebugFlags=TraceJobs` 和 `SlurmctldDebug=verbose`（或更高等級）。

---

### 作業狀態 (Job States)

系統中每個已知的作業都會具有以下其中一個狀態：

| 名稱 | 說明 |
|------|------|
| `BOOT_FAIL` | 因節點開機失敗而終止 |
| `CANCELLED` | 被使用者或管理員取消 |
| `COMPLETED` | 成功完成執行；在所有節點上以[結束代碼](job_exit_code.md)零結束 |
| `DEADLINE` | 因達到最晚開始時間而終止（該時間是根據 TimeLimit 計算出能讓作業在期限內完成的最晚開始點） |
| `FAILED` | 執行未成功完成；非零[結束代碼](job_exit_code.md)或其他失敗條件 |
| `NODE_FAIL` | 因節點故障而終止 |
| `OUT_OF_MEMORY` | 發生記憶體不足錯誤 |
| `PENDING` | 已排入佇列等待啟動；通常會有一個[原因代碼](job_reason_codes.md)說明為何尚未開始 |
| `PREEMPTED` | 因[搶佔](../administrators/preempt.md)而終止；可能根據設定的 PreemptMode 和作業特性轉換到另一個狀態 |
| `RUNNING` | 已分配資源且正在執行 |
| `SUSPENDED` | 已分配資源但執行被暫停，例如因[搶佔](../administrators/preempt.md)或授權使用者的[直接請求](scontrol.html#OPT_suspend) |
| `TIMEOUT` | 因達到時間限制而終止，例如在 [slurm.conf](slurm.conf.html) 中設定的時間或為個別作業指定的時間 |

---

### 作業旗標 (Job Flags)

作業可能會設置額外的旗標：

| 名稱 | 說明 |
|------|------|
| `COMPLETING` | 作業已完成或已取消，正在執行清理任務，包括 [epilog](prolog_epilog.md) 腳本（如果存在） |
| `CONFIGURING` | 作業已分配節點，正在等待節點開機或重新開機 |
| `EXPEDITING` | 作業立即具有最高優先順序的排程資格 |
| `LAUNCH_FAILED` | 在選定的節點上啟動失敗；包括 [prolog](prolog_epilog.md) 失敗和其他失敗條件 |
| `POWER_UP_NODE` | 作業已分配到關機狀態的節點，正在等待節點開機 |
| `RECONFIG_FAIL` | 作業的節點設定失敗 |
| `REQUEUED` | 作業正在重新排入佇列，例如因[搶佔](../administrators/preempt.md)或授權使用者的[直接請求](scontrol.html#OPT_requeue) |
| `REQUEUE_FED` | 因[聯邦](federation.md)設定中兄弟作業的條件而重新排入佇列 |
| `REQUEUE_HOLD` | 與 `REQUEUED` 相同，但在被[釋放](scontrol.html#OPT_release)之前不會被考慮排程 |
| `RESIZING` | 作業的大小正在改變；防止發生衝突的作業變更 |
| `RESV_DEL_HOLD` | 因預約被刪除而保留 |
| `REVOKED` | 因[聯邦](federation.md)設定中兄弟作業的條件而被撤銷 |
| `SIGNALING` | 正在等待向作業發送信號 |
| `SPECIAL_EXIT` | 與 `REQUEUE_HOLD` 相同，但用於識別適用於此作業的[特殊情況](scontrol.html#OPT_State) |
| `STAGE_OUT` | 正在輸出資料（[burst buffer](burst_buffer.md)） |
| `STOPPED` | 收到 SIGSTOP 信號以暫停作業而不釋放資源 |
| `UPDATE_DB` | 正在將作業更新資訊發送到資料庫 |

---

## 說明

### 基本狀態 vs 狀態旗標

- **基本狀態 (Base State)**：每個作業都有且僅有一個基本狀態，代表作業的主要生命週期階段（等待、執行中、完成等）
- **狀態旗標 (State Flag)**：提供關於作業目前正在進行什麼操作的額外資訊。一個作業可以同時有多個旗標

### 狀態顯示邏輯

當使用 `squeue` 或 `sacct` 查看作業時：
1. 如果有可識別的旗標，顯示旗標
2. 否則顯示基本狀態
3. REST API 會同時回傳基本狀態和所有旗標

### 終止狀態分類

| 類型 | 狀態 | 常見原因 |
|------|------|----------|
| 成功終止 | `COMPLETED` | 正常完成，結束代碼為 0 |
| 失敗終止 | `FAILED` | 程式錯誤、非零結束代碼 |
| 系統終止 | `TIMEOUT`, `NODE_FAIL`, `BOOT_FAIL` | 超時、節點故障 |
| 人為終止 | `CANCELLED`, `PREEMPTED` | 使用者取消、被搶佔 |
| 資源問題 | `OUT_OF_MEMORY`, `DEADLINE` | 記憶體不足、錯過期限 |

---

## 實務範例

### 查看作業狀態

```bash
# 查看所有作業狀態
squeue -o "%.10i %.9P %.20j %.8u %.8T %.10M %.9l %.6D %R"

# 查看特定狀態的作業
squeue -t PENDING          # 只顯示等待中的作業
squeue -t RUNNING          # 只顯示執行中的作業
squeue -t COMPLETING       # 只顯示正在清理的作業

# 查看所有非執行狀態的作業
squeue -t PENDING,SUSPENDED,CONFIGURING
```

### 使用 sacct 查看歷史狀態

```bash
# 查看今天的所有作業及其狀態
sacct -S today -o JobID,JobName,State,ExitCode

# 查看失敗的作業
sacct -S today --state=FAILED,TIMEOUT,OUT_OF_MEMORY

# 查看特定作業的完整狀態歷史
sacct -j 12345 --format=JobID,State,Start,End,ExitCode
```

### 監控作業狀態變化

```bash
# 每 5 秒監控一次作業狀態
watch -n 5 'squeue -j 12345 -o "%.10i %.8T %.10M"'

# 啟用詳細除錯來追蹤狀態變化（管理員）
# 在 slurm.conf 中設定：
# DebugFlags=TraceJobs
# SlurmctldDebug=verbose
```

### 根據狀態採取行動

```bash
# 取消所有 PENDING 超過 24 小時的作業
for job in $(squeue -u $USER -t PENDING -h -o "%i %V" | \
    awk -v d=$(date -d "yesterday" +%Y-%m-%dT%H:%M:%S) '$2 < d {print $1}'); do
    scancel $job
done

# 重新提交 TIMEOUT 的作業
sacct -u $USER -S today --state=TIMEOUT -n -o JobID | while read jobid; do
    scontrol requeue $jobid
done
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 混淆 COMPLETING 和 COMPLETED | COMPLETING 是正在清理，COMPLETED 是已完成 |
| 忽略 OUT_OF_MEMORY 狀態 | 檢查記憶體請求是否足夠，考慮使用 `--mem` |
| 不理解 REQUEUE_HOLD | 需要手動釋放：`scontrol release <jobid>` |
| 對 CONFIGURING 等太久 | 可能是節點開機問題，聯繫管理員 |

### 狀態診斷技巧

1. **PENDING 太久**：使用 `squeue -j <jobid> -o "%r"` 查看原因代碼
2. **FAILED 原因不明**：使用 `sacct -j <jobid> --format=State,ExitCode,DerivedExitCode`
3. **COMPLETING 停滯**：可能是 epilog 腳本問題，聯繫管理員
4. **反覆 REQUEUED**：檢查是否有搶佔設定或節點問題

### 預防措施

- 設定合理的時間限制避免 TIMEOUT
- 使用 `--mem` 和 `--mem-per-cpu` 防止 OUT_OF_MEMORY
- 在腳本中檢查退出碼，確保正確回報狀態
- 使用 `--mail-type=ALL` 接收狀態變更通知

---

## 快速參考

### 基本狀態速查表

| 代碼 | 縮寫 | 說明 | 建議動作 |
|------|------|------|----------|
| BOOT_FAIL | BF | 節點開機失敗 | 聯繫管理員 |
| CANCELLED | CA | 被取消 | 檢查取消原因 |
| COMPLETED | CD | 成功完成 | 無 |
| DEADLINE | DL | 錯過期限 | 調整提交時間或 TimeLimit |
| FAILED | F | 執行失敗 | 檢查結束代碼和日誌 |
| NODE_FAIL | NF | 節點故障 | 聯繫管理員，考慮重新提交 |
| OUT_OF_MEMORY | OOM | 記憶體不足 | 增加 `--mem` 設定 |
| PENDING | PD | 等待中 | 查看原因代碼 |
| PREEMPTED | PR | 被搶佔 | 可能自動重排或需重新提交 |
| RUNNING | R | 執行中 | 監控進度 |
| SUSPENDED | S | 被暫停 | 等待恢復或釋放 |
| TIMEOUT | TO | 超時 | 增加時間限制或優化程式 |

### 旗標速查表

| 旗標 | 說明 | 處理方式 |
|------|------|----------|
| COMPLETING | 清理中 | 等待 epilog 完成 |
| CONFIGURING | 設定節點 | 等待節點開機 |
| REQUEUED | 重新排入 | 將自動重新排程 |
| REQUEUE_HOLD | 保留重排 | 需手動 `scontrol release` |
| RESV_DEL_HOLD | 預約刪除保留 | 需手動釋放或重新提交 |
