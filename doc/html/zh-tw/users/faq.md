# Slurm 常見問題 (FAQ)

## TL;DR
- 本文涵蓋管理者、研究人員和使用者的常見問題
- 包含作業設計、提交、排程、管理和錯誤訊息的解答
- 遇到問題時先查看作業狀態和原因：`scontrol show job <jobid>`
- 系統管理員問題涵蓋配置、除錯、叢集管理和資料庫

---

## 管理層問題

### Slurm 真的是免費的嗎？

是的，Slurm 是免費且開源的：
- Slurm 符合[自由軟體基金會](https://www.gnu.org/philosophy/free-sw.en.html)對自由軟體的定義
- Slurm 的[原始碼](https://github.com/SchedMD/slurm)和[文件](https://slurm.schedmd.com/documentation.html)在 GNU GPL v2 下公開提供
- Slurm 可以免費下載、使用、修改和重新分發

### 為什麼要使用 Slurm 或其他自由軟體？

自由軟體與專有軟體一樣，品質參差不齊，但這種機制已被證明能夠產生被全球公司信任的高品質軟體。Linux 核心就是一個突出的例子。

同樣，自 2002 年首次發布以來，Slurm 已成為超級計算領域值得信賴的工具。如今，Slurm 為大多數 [TOP500](https://www.top500.org/) 超級電腦提供動力。從商業工作負載管理器切換到 Slurm 的客戶通常報告更高的可擴展性、更好的效能和更低的成本。

### "Slurm" 代表什麼？

什麼都不代表。最初，「SLURM」（完全大寫）是「Simple Linux Utility for Resource Management」的縮寫。2012 年，首選大寫改為 Slurm，並且放棄了縮寫。

---

## 研究人員問題

### 如何引用涉及 Slurm 的工作？

我們建議引用 JSSPP 2023 的同儕評審論文：
[Architecture of the Slurm Workload Manager](https://doi.org/10.1007/978-3-031-43943-8_1)

```
Jette, M.A., Wickberg, T. (2023). Architecture of the Slurm Workload Manager.
In: Klusáček, D., Corbalán, J., Rodrigo, G.P. (eds) Job Scheduling Strategies
for Parallel Processing. JSSPP 2023. Lecture Notes in Computer Science,
vol 14283. Springer, Cham. https://doi.org/10.1007/978-3-031-43943-8_1
```

---

## 使用者問題

### 設計作業

#### 如何在單一腳本中執行多個作業？

Slurm 作業只是一個資源分配。您可以在該分配中執行許多作業步驟，可以是平行或順序的。有些作業實際上以這種方式啟動數千個作業步驟。作業步驟將被分配尚未分配給其他作業步驟的節點。

#### CPU、核心和執行緒的區別是什麼？

- 如果您的節點配置了超執行緒 (hyperthreading)，則 CPU 等同於一個超執行緒
- 否則 CPU 等同於一個核心
- 您可以使用 `scontrol show node` 指令並查看「ThreadsPerCore」值來確定

即使在啟用超執行緒的系統上，資源通常也會在核心級別分配給作業。兩個不同的作業不會共享一個核心，除非使用分割區的 OverSubscribe 配置參數。

#### 如何在分配中的特定節點上執行特定任務？

使用 srun 的分發方法之一 '-m' 或 '--distribution' 是 'arbitrary'。這表示您可以告訴 Slurm 以任何您想要的方式佈局您的任務：

```bash
# 在第一個節點上執行 4 個任務，第二個節點上執行 1 個任務
srun -n5 -m arbitrary -w tux[0,0,0,0,1] hostname
```

#### Slurm 如何為我的作業建立環境？

Slurm 程序不是在 shell 下執行的，而是由 slurmd 守護程序直接 exec。執行 srun 指令時生效的環境變數會傳播到產生的程序。~/.profile 和 ~/.bashrc 腳本不會作為程序啟動的一部分執行。

### 提交作業

#### 為什麼我的 srun 選項被忽略？

srun 指令之後的所有內容都會被檢查以確定它是否是 srun 的有效選項。第一個不是 srun 有效選項的標記被視為要執行的指令，之後的所有內容都被視為該指令的選項。

```bash
# 錯誤：-pdebug 被視為 uptime 的選項
srun -N2 uptime -pdebug

# 正確：選項應該在指令之前
srun -N2 -pdebug uptime
```

#### sbatch 和 srun 指令有什麼區別？

**srun** 指令設計用於互動式使用，有人監控輸出。應用程式的輸出會顯示在使用者的終端機上。

**sbatch** 指令設計用於稍後執行提交的腳本，其輸出寫入檔案。

主要差異：
- sbatch 支援[作業陣列](job_array.md)的概念，而 srun 不支援
- sbatch 作業的故障通常會導致作業重新排隊並再次執行
- srun 的故障通常會生成錯誤訊息

#### 如何在互動模式下獲得 shell 提示？

從 20.11 開始，獲得互動式 shell 提示的推薦方法是在 slurm.conf 中配置：

```
LaunchParameters=use_interactive_step
```

這會配置 salloc 在沒有指定要執行的程式時，自動透過 srun 在分配中的節點上啟動互動式 shell。

#### Slurm 可以在分配的計算節點上匯出 X11 顯示嗎？

從 17.11 版本開始，您可以使用內建的 X11 功能。在 slurm.conf 中設定 `PrologFlags=x11` 即可啟用：

```bash
$ ssh -X user@login1
$ srun -n1 --pty --x11 xclock
```

### 排程

#### 為什麼我的作業沒有執行？

答案取決於許多因素。主要因素是 Slurm 使用哪種排程器：

```bash
scontrol show config | grep SchedulerType
```

- **builtin**：作業將按照提交順序執行
- **backfill**：作業通常按提交順序執行，但稍後提交的作業如果不延遲較早提交作業的預期執行時間，可以提前啟動

作為指南，發出 `scontrol show job <jobid>` 並查看 State 和 Reason 欄位來調查原因。

#### 為什麼 Slurm 回填排程器不啟動我的作業？

最常見的問題是**未設定作業時間限制**。如果所有作業都有相同的時間限制（例如分割區的時間限制），則回填將無效。

建議：
- 分割區可以有預設和最大時間限制
- 使用者應為作業指定合理的時間限制

### 被終止的作業

#### 為什麼我的作業過早被終止？

Slurm 有一個作業清除機制，用於在達到其時間限制之前移除不活動的作業。此不活動時間限制可由系統管理員配置：

```bash
scontrol show config | grep InactiveLimit
```

InactiveLimit 的值以秒為單位。零值表示作業清除被禁用。

#### "srun: Force Terminated job" 表示什麼？

srun 指令通常在產生的任務的標準輸出和錯誤 I/O 結束時終止。當作業步驟終止時，無論是達到其時間限制還是被明確終止，srun 都會收到通知。如果 srun 尚未終止，則會列印「srun: Force Terminated job」訊息。

#### "srun: First task exited 30s ago" 然後 "srun Job Failed" 是什麼意思？

srun 指令監控任務何時退出。預設情況下，第一個任務退出後 30 秒，作業將被終止。這通常表示某種類型的作業失敗。可以使用 srun 的 `--wait=<time>` 選項更改此行為。

### 管理作業

#### 如何暫時阻止作業執行（保持狀態）？

最簡單的方法是更改作業的最早開始時間：

```bash
# 將作業置於保持狀態（30 天內無法啟動）
$ scontrol update JobId=1234 StartTime=now+30days

# 稍後允許它立即開始
$ scontrol update JobId=1234 StartTime=now
```

#### 作業開始執行後可以更改其大小嗎？

Slurm 支援減小作業大小的能力。使用 scontrol 指令透過指定新的節點數（NumNodes=）或識別您希望作業保留的特定節點（NodeList=）來更改作業的大小。

```bash
#!/bin/bash
srun my_big_job
scontrol update JobId=$SLURM_JOB_ID NumNodes=2
. slurm_job_${SLURM_JOB_ID}_resize.sh
srun -N2 my_small_job
```

#### 為什麼我的作業/節點處於 COMPLETING 狀態？

當作業終止時，作業及其節點都會進入 COMPLETING 狀態。通常，這會在一秒內發生。但是，如果作業有無法使用 SIGKILL 訊號終止的程序，作業和一個或多個節點可能會長時間保持在 COMPLETING 狀態。

解決方案：
```bash
# 將節點設為 DOWN
scontrol update NodeName=<name> State=DOWN Reason=hung_completing

# 重新啟動節點後，重設狀態
scontrol update NodeName=<name> State=RESUME
```

#### 如何重新排隊已完成或失敗狀態的作業？

使用指令：
```bash
scontrol requeue job_id
```

作業將被重新排隊回到 PENDING 狀態並再次排程。

要重新排隊到保持狀態：
```bash
scontrol requeuehold job_id
```

### 資源限制

#### 為什麼我的資源限制沒有傳播？

可能的原因：
1. 應用於 slurmd 守護程序的硬資源限制低於使用者在提交主機上的軟資源限制
2. 分配節點上使用者的硬資源限制低於提交作業的節點上同一使用者的軟硬資源限制
3. slurm.conf 中配置了 PropagateResourceLimits 或 PropagateResourceLimitsExcept 參數

#### 為什麼我的 MPI 作業因為鎖定記憶體限制太低而失敗？

預設情況下，Slurm 在作業提交時將所有資源限制傳播到產生的任務。可以透過在 slurm.conf 檔案中排除特定限制的傳播來禁用此功能：

```
PropagateResourceLimitsExcept=MEMLOCK
```

---

## 系統管理員問題

### 測試環境

#### 可以並行執行多個 Slurm 系統進行測試嗎？

是的，使用不同的配置檔案和狀態儲存目錄即可。

#### Slurm 可以模擬更大的叢集嗎？

是的。使用多個 slurmd 守護程序在單一節點上配置（每個都有不同的 NodeName）。

### 建置和安裝

#### 為什麼 pam_slurm.so 或其他組件不在 Slurm RPM 中？

一些組件是可選的，可能需要額外的依賴項。需要手動從原始碼建置。

#### 如何使用除錯符號建置 Slurm？

```bash
./configure --enable-debug
```

#### 如何從 Slurm 的 GitHub commit 生成 patch 檔案？

```bash
git format-patch -1 <commit-hash>
```

#### 如何將 patch 應用到我的 Slurm 原始碼？

```bash
git apply <patch-file>
# 或
patch -p1 < <patch-file>
```

### 叢集管理

#### 如何重新定位主要或備份控制器？

1. 更新 slurm.conf 中的 SlurmctldHost
2. 將配置分發到所有節點
3. 重新啟動 slurmctld 守護程序

#### 我需要在叢集上維護同步時鐘嗎？

是的，Slurm 需要所有節點上的時鐘同步。建議使用 NTP 或類似服務。

#### 如何停止 Slurm 排程作業？

```bash
# 停止排程但允許現有作業執行
scontrol reconfig
# 在 slurm.conf 中設定 SchedulerType=builtin

# 或使用
scontrol update PartitionName=<name> State=INACTIVE
```

#### 如何為維護期間排空工作負載？

1. 建立預約：
```bash
scontrol create reservation starttime=<time> duration=<minutes> \
    user=root nodes=ALL flags=maint,ignore_jobs
```

2. 或將分割區設為 DRAIN：
```bash
scontrol update PartitionName=<name> State=DRAIN
```

#### 升級 Slurm 時應該注意什麼？

- 閱讀 RELEASE_NOTES
- 先升級 slurmdbd，然後是 slurmctld，最後是 slurmd
- 保持版本一致性（不超過兩個主要版本差異）

### 計算節點

#### 為什麼節點在註冊服務後顯示為 DOWN 狀態？

如果 ReturnToService 配置為 0 或 1，需要管理員手動將節點恢復：

```bash
scontrol update NodeName=<name> State=RESUME
```

設定 ReturnToService=2 將自動恢復節點。

#### 當節點崩潰時會發生什麼？

在該節點上執行的作業會失敗。如果配置了 JobRequeue=1，作業可能會自動重新排隊。

#### 如何向 Slurm 添加節點？

1. 在 slurm.conf 中添加節點配置
2. 將配置分發到所有節點
3. 重新配置：`scontrol reconfigure`
4. 在新節點上啟動 slurmd

#### 如何從 Slurm 移除節點？

1. 排空節點上的作業：`scontrol update NodeName=<name> State=DRAIN`
2. 等待所有作業完成
3. 從 slurm.conf 中移除節點
4. 重新配置：`scontrol reconfigure`

### 使用者管理

#### PAM 可以用來控制使用者對計算節點的存取嗎？

是的，Slurm 提供 pam_slurm_adopt 模組，可以限制只有在節點上有作業執行的使用者才能 SSH 到該節點。

### 錯誤訊息

#### "Credential replayed" 在 SlurmdLogFile 中

這表示相同的作業憑證被使用了兩次。可能的原因：
- 節點時鐘不同步
- 作業重新提交時有問題

#### "Invalid job credential"

可能的原因：
- slurmctld 和 slurmd 使用不同的 SlurmsparamKey
- 節點時鐘不同步

#### "Unable to accept new connection: Too many open files"

增加檔案描述符限制：
```bash
ulimit -n 65536
```

或在 systemd 服務檔案中設定 LimitNOFILE。

---

## Explanation

### 常見作業狀態

| 狀態 | 說明 |
|------|------|
| PD | Pending - 待處理，等待資源 |
| R | Running - 執行中 |
| CG | Completing - 正在完成 |
| CD | Completed - 已完成 |
| F | Failed - 失敗 |
| TO | Timeout - 超時 |
| CA | Cancelled - 被取消 |
| S | Suspended - 被暫停 |

### 常見待處理原因

| 原因 | 說明 |
|------|------|
| Resources | 等待資源 |
| Priority | 等待更高優先級 |
| Dependency | 等待依賴作業完成 |
| QOSMaxJobsPerUserLimit | 超過 QOS 使用者作業數限制 |
| ReqNodeNotAvail | 請求的節點不可用 |
| PartitionDown | 分割區停用 |

---

## Practical Example

### 常見問題診斷流程

```bash
# 1. 查看作業狀態和原因
$ scontrol show job <jobid> | grep -E "State|Reason"
   JobState=PENDING Reason=Resources

# 2. 查看分割區狀態
$ sinfo -p <partition>

# 3. 查看節點狀態
$ scontrol show node <nodename>

# 4. 查看作業詳細資訊
$ scontrol show job <jobid>

# 5. 查看排程器配置
$ scontrol show config | grep -E "Scheduler|Backfill"

# 6. 查看資源限制
$ sacctmgr show qos
$ sacctmgr show assoc user=<username>
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **忘記設定作業時間限制**
   ```bash
   # 問題：回填排程無效
   # 所有作業都使用分割區預設時間限制

   # 解決：設定合理的時間限制
   sbatch -t 2:00:00 my_job.sh
   ```

2. **srun 選項位置錯誤**
   ```bash
   # 錯誤
   srun hostname -N2

   # 正確
   srun -N2 hostname
   ```

3. **互動式作業設定問題**
   ```bash
   # 舊方法（可能導致問題）
   srun --pty bash -i

   # 推薦方法（20.11+）
   # 在 slurm.conf 設定 LaunchParameters=use_interactive_step
   salloc -N1
   ```

### 實用建議

- **定期檢查作業狀態**：使用 `squeue -u $USER` 監控您的作業
- **使用合理的時間限制**：讓回填排程更有效
- **指定資源需求**：明確的資源請求有助於排程
- **檢查錯誤日誌**：`~/.slurm/job_<jobid>.err` 或指定的錯誤檔案
- **使用作業陣列**：處理大量相似作業時更高效

---

## Quick Reference

### 常用診斷指令

| 指令 | 說明 |
|------|------|
| `squeue -u $USER` | 查看您的作業 |
| `scontrol show job <id>` | 查看作業詳情 |
| `sinfo` | 查看分割區和節點狀態 |
| `scontrol show config` | 查看 Slurm 配置 |
| `sacct -j <id>` | 查看作業會計資訊 |
| `sshare -u <user>` | 查看使用者份額 |

### 作業管理指令

| 指令 | 說明 |
|------|------|
| `scontrol hold <id>` | 保持作業 |
| `scontrol release <id>` | 釋放作業 |
| `scontrol requeue <id>` | 重新排隊作業 |
| `scancel <id>` | 取消作業 |
| `scontrol update JobId=<id> ...` | 更新作業 |

### 環境變數

| 變數 | 說明 |
|------|------|
| `SLURM_JOB_ID` | 作業 ID |
| `SLURM_JOB_NODELIST` | 分配的節點列表 |
| `SLURM_NTASKS` | 任務數量 |
| `SLURM_CPUS_PER_TASK` | 每任務 CPU 數 |
| `SLURM_JOB_PARTITION` | 分割區名稱 |
