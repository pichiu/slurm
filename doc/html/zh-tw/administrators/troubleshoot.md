# Slurm 疑難排解指南

---

## TL;DR

本指南幫助系統管理員排解 Slurm 故障。常見問題包括：Slurm 無回應（檢查 `scontrol ping`、slurmctld 狀態）、作業未排程（檢查排程器類型、作業優先權）、作業/節點卡在 COMPLETING（通常是檔案系統問題）、節點變成 DOWN（檢查 `scontrol show node`、slurmd 狀態）、網路問題（檢查日誌、配置一致性）。

---

## Translation（翻譯）

### 目錄

- [Slurm 無回應](#slurm-無回應)
- [作業未被排程](#作業未被排程)
- [作業和節點卡在 COMPLETING 狀態](#作業和節點卡在-completing-狀態)
- [節點被設為 DOWN 狀態](#節點被設為-down-狀態)
- [網路和配置問題](#網路和配置問題)

本指南旨在作為幫助系統管理員或操作員排解 Slurm 故障並恢復服務的工具。常見問題文件也可能有所幫助。

### Slurm 無回應

1. 執行 `scontrol ping` 以確定主控制器和備援控制器是否有回應。

2. 如果它對您有回應，這可能是叢集中某個使用者或節點特有的網路或配置問題。

3. 如果沒有回應，直接登入該機器並再試一次，以排除網路和配置問題。

4. 如果仍然沒有回應，透過執行 `ps -el | grep slurmctld` 檢查是否有活躍的 slurmctld 常駐程式。

5. 如果 slurmctld 未執行，請重新啟動它（通常以 root 使用者使用指令 `/etc/init.d/slurm start`）。您應該檢查日誌檔案（slurm.conf 檔案中的 `SlurmctldLog`）以了解失敗原因。

6. 如果 slurmctld 正在執行但沒有回應（非常罕見的情況），則終止並重新啟動它（通常以 root 使用者使用指令 `/etc/init.d/slurm stop` 然後 `/etc/init.d/slurm start`）。

7. 如果它再次掛起，增加除錯訊息的詳細程度（在 slurm.conf 檔案中增加 `SlurmctldDebug`）並重新啟動。再次檢查日誌檔案以了解失敗原因。

8. 如果它繼續失敗而沒有指示故障模式，則在不保留狀態的情況下重新啟動（通常以 root 使用者使用指令 `/etc/init.d/slurm stop` 然後 `/etc/init.d/slurm startclean`）。**注意**：所有執行中的作業和其他狀態資訊將會遺失。

### 作業未被排程

這取決於 Slurm 使用的排程器。執行指令 `scontrol show config | grep SchedulerType` 來確定這一點。對於任何排程器，您可以使用指令 `scontrol show job` 檢查作業的優先權。

**排程器類型：**

| 排程器類型 | 行為 |
|-----------|------|
| **builtin** | 作業將按給定分割區的提交順序執行。即使有資源可立即啟動作業，它也會延遲直到沒有先前提交的作業處於待處理狀態 |
| **backfill** | 作業通常按給定分割區的提交順序執行，但有一個例外：如果這樣做不會延遲較早提交作業的預期執行時間，則稍後提交的作業會提前啟動 |

**回填排程的效能考量：**
- 為了使回填排程有效，使用者的作業應指定合理的時間限制
- 如果作業未指定時間限制，則所有作業將獲得相同的時間限制（與分割區相關的時間限制），回填排程作業的能力將受到限制
- 回填排程器不會更改所需或排除節點的作業規格，因此指定節點的作業將大大降低回填排程的效率

### 作業和節點卡在 COMPLETING 狀態

這通常是由於與作業相關的不可終止程序造成的。Slurm 將繼續嘗試使用 SIGKILL 終止程序，但某些作業可能卡在執行 I/O 而無法終止。這通常是由於檔案系統問題，可以透過以下幾種方式解決：

1. **修復檔案系統和/或重新啟動節點** -或-

2. **將節點設為 DOWN 狀態然後恢復服務**
   ```bash
   scontrol update NodeName=<node> State=down Reason=hung_proc
   scontrol update NodeName=<node> State=resume
   ```
   這允許其他作業使用該節點，但保留不可終止的程序。如果該程序應該完成 I/O，待處理的 SIGKILL 應該立即終止它。-或-

3. **使用 UnkillableStepProgram 和 UnkillableStepTimeout** 配置參數自動回應無法終止的程序，透過發送電子郵件或重新啟動節點。

**除錯步驟：**

如果看起來作業不是因為檔案系統問題而卡住，可能需要一些除錯來找到原因。如果可以重現該行為，可以將 SlurmdDebug 等級設為 'debug' 並在將用於重現問題的節點上重新啟動 slurmd。然後 slurmd.log 檔案應該有更多資訊來幫助排解問題。

查看 slurmctld.log 也可能提供線索。如果節點停止回應，您可能需要調查原因，因為它們可能會阻止作業清理並導致作業保持在 COMPLETING 狀態。查找連線問題時，相關的日誌條目應該如下所示：

```
error: Nodes node[00,03,25] not responding
Node node00 now responding
```

### 節點被設為 DOWN 狀態

1. 使用指令 `scontrol show node <name>` 檢查節點為何 DOWN。這將顯示節點被設為 DOWN 的原因以及發生的時間。如果與 slurm.conf 檔案中指定的參數相比磁碟空間、記憶體空間等不足，則修復節點或更改 slurm.conf。

2. 如果原因是「Not responding」，則使用指令 `ping <address>` 檢查控制機器和 DOWN 節點之間的通訊，確保指定 slurm.conf 中配置的 NodeAddr 值。如果 ping 失敗，則修復網路或 slurm.conf 中的地址。

3. 接下來，登入 Slurm 認為處於 DOWN 狀態的節點，並使用指令 `ps -el | grep slurmd` 檢查 slurmd 常駐程式是否正在執行。如果 slurmd 未執行，請重新啟動它（通常以 root 使用者使用指令 `/etc/init.d/slurm start`）。您應該檢查日誌檔案（slurm.conf 檔案中的 `SlurmdLog`）以了解失敗原因。您可以透過在相關節點上執行指令 `scontrol show slurmd` 來獲取執行中 slurmd 常駐程式的狀態。檢查「Last slurmctld msg time」的值以確定 slurmctld 是否能夠與 slurmd 通訊。

4. 如果 slurmd 正在執行但沒有回應（非常罕見的情況），則終止並重新啟動它（通常以 root 使用者使用指令 `/etc/init.d/slurm stop` 然後 `/etc/init.d/slurm start`）。

5. 如果仍然沒有回應，再試一次以排除網路和配置問題。

6. 如果仍然沒有回應，增加除錯訊息的詳細程度（在 slurm.conf 檔案中增加 `SlurmdDebug`）並重新啟動。再次檢查日誌檔案以了解失敗原因。

7. 如果仍然沒有回應且沒有指示故障模式，則在不保留狀態的情況下重新啟動（通常以 root 使用者使用指令 `/etc/init.d/slurm stop` 然後 `/etc/init.d/slurm startclean`）。**注意**：該節點上的所有作業和其他狀態資訊將會遺失。

### 網路和配置問題

1. 檢查控制器和/或 slurmd 日誌檔案（slurm.conf 檔案中的 `SlurmctldLog` 和 `SlurmdLog`）以了解失敗原因。

2. 檢查發生問題的節點上 slurm.conf 和憑證檔案的一致性。

3. 如果這是特定使用者的問題，檢查該使用者是否在控制器電腦以及運算節點上都有配置。使用者不需要能夠登入，但其使用者 ID 必須存在。

4. 檢查所有節點上是否存在相容版本的 Slurm（執行 `sinfo -V` 或 `rpm -qa | grep slurm`）。Slurm 版本號包含三個以句點分隔的數字，代表主要 Slurm 版本和維護版本等級。前兩部分組合代表主要版本，並與該主要版本的年份和月份匹配。版本中的第三個數字表示特定的維護等級：year.month.maintenance-release（例如 17.11.5 是主要 Slurm 版本 17.11，維護版本 5）。因此版本 17.11.x 最初於 2017 年 11 月發布。Slurm 常駐程式將支援來自前兩個主要版本的 RPC 和狀態檔案。

---

## Explanation（解釋）

### 故障排解流程圖

```
問題發生
    │
    ▼
┌─────────────────────────────────────────┐
│           Slurm 無回應？                 │
│                                         │
│  1. scontrol ping                       │
│  2. 檢查 slurmctld 程序                 │
│  3. 檢查日誌檔案                        │
│  4. 增加除錯等級                        │
│  5. 重新啟動（必要時清除狀態）          │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│           節點 DOWN？                    │
│                                         │
│  1. scontrol show node <name>           │
│  2. 檢查 DOWN 原因                      │
│  3. 檢查 slurmd 程序                    │
│  4. 檢查網路連線                        │
│  5. 檢查配置一致性                      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│         作業卡在 COMPLETING？            │
│                                         │
│  1. 檢查不可終止程序                    │
│  2. 檢查檔案系統問題                    │
│  3. 設定節點 DOWN 再 RESUME             │
│  4. 使用 UnkillableStepProgram          │
└─────────────────────────────────────────┘
```

### 常見狀態和含義

| 狀態 | 含義 | 可能原因 |
|------|------|----------|
| `DOWN` | 節點離線 | slurmd 未執行、網路問題、資源不足 |
| `DOWN*` | 節點無回應 | 通訊失敗、slurmd 掛起 |
| `DRAIN` | 節點排空中 | 管理員手動設定、錯誤 |
| `COMPLETING` | 作業完成中 | 程序清理中、I/O 阻塞 |
| `FAIL` | 節點失敗 | 硬體故障、配置錯誤 |

---

## Practical Example（實用範例）

### 範例 1：診斷無回應的 Slurm

```bash
# 步驟 1：檢查控制器狀態
scontrol ping
# 輸出：Slurmctld(primary) at server1 is UP

# 步驟 2：如果無回應，檢查程序
ssh server1 "ps aux | grep slurmctld"

# 步驟 3：檢查日誌
ssh server1 "tail -100 /var/log/slurm/slurmctld.log"

# 步驟 4：重新啟動服務
ssh server1 "systemctl restart slurmctld"
```

### 範例 2：診斷 DOWN 節點

```bash
# 步驟 1：查看節點狀態
scontrol show node node001

# 輸出範例：
# NodeName=node001 State=DOWN* Reason=Not responding
# ...

# 步驟 2：測試網路連線
ping node001

# 步驟 3：登入節點檢查 slurmd
ssh node001 "systemctl status slurmd"

# 步驟 4：檢查 slurmd 日誌
ssh node001 "tail -50 /var/log/slurm/slurmd.log"

# 步驟 5：重新啟動 slurmd
ssh node001 "systemctl restart slurmd"

# 步驟 6：恢復節點
scontrol update NodeName=node001 State=resume
```

### 範例 3：處理 COMPLETING 狀態

```bash
# 查看卡住的作業
squeue -t COMPLETING

# 查看節點上的程序
ssh node001 "ps aux | grep <job_id>"

# 方法 1：強制終止（如果安全）
scancel -f <job_id>

# 方法 2：設定節點 DOWN 再恢復
scontrol update NodeName=node001 State=down Reason="clearing stuck job"
scontrol update NodeName=node001 State=resume

# 方法 3：重新啟動節點（最後手段）
ssh node001 "reboot"
```

### 範例 4：檢查配置一致性

```bash
# 檢查所有節點的 Slurm 版本
clush -a "sinfo -V"

# 檢查配置檔案是否相同
clush -a "md5sum /etc/slurm/slurm.conf"

# 檢查 MUNGE 金鑰
clush -a "md5sum /etc/munge/munge.key"
```

### 範例 5：增加除錯等級

```bash
# 臨時增加 slurmctld 除錯等級
scontrol setdebug debug3

# 或修改配置檔（需要重新載入）
# slurm.conf:
# SlurmctldDebug=debug3

scontrol reconfigure

# 查看除錯日誌
tail -f /var/log/slurm/slurmctld.log
```

### 範例 6：作業未排程診斷

```bash
# 檢查排程器類型
scontrol show config | grep SchedulerType

# 查看作業優先權
sprio -j <job_id>

# 查看作業詳情
scontrol show job <job_id>

# 查看作業等待原因
squeue -j <job_id> --format="%i %r"

# 檢查資源可用性
sinfo -N -l

# 檢查排程器診斷
sdiag
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 後果 | 解決方案 |
|------|------|----------|
| 直接終止 slurmctld | 可能遺失狀態 | 使用正確的停止指令 |
| 忽略日誌檔案 | 錯過重要錯誤訊息 | 定期檢查日誌 |
| 配置不一致 | 通訊失敗 | 確保所有節點配置同步 |
| MUNGE 金鑰不同 | 認證失敗 | 同步所有節點的金鑰 |
| 版本不相容 | RPC 失敗 | 保持版本在兩個主版本內 |

### ✅ 實用技巧

1. **設定日誌輪轉**
   ```bash
   # /etc/logrotate.d/slurm
   /var/log/slurm/*.log {
       weekly
       rotate 4
       compress
       missingok
       notifempty
   }
   ```

2. **監控關鍵指標**
   ```bash
   # 監控節點狀態
   watch -n 10 'sinfo -R'

   # 監控作業佇列
   watch -n 10 'squeue -t PD | head -20'
   ```

3. **自動化健康檢查**
   ```bash
   #!/bin/bash
   # 健康檢查腳本
   if ! scontrol ping > /dev/null 2>&1; then
       echo "WARNING: slurmctld not responding"
       # 發送警報
   fi

   down_nodes=$(sinfo -h -t down -o "%n" | wc -l)
   if [ $down_nodes -gt 0 ]; then
       echo "WARNING: $down_nodes nodes are down"
   fi
   ```

4. **保存故障前狀態**
   ```bash
   # 在重新啟動前保存狀態
   scontrol show config > /tmp/slurm_config_$(date +%Y%m%d).txt
   scontrol show node > /tmp/slurm_nodes_$(date +%Y%m%d).txt
   scontrol show job > /tmp/slurm_jobs_$(date +%Y%m%d).txt
   ```

---

## Quick Reference（快速參考）

### 診斷指令

| 指令 | 功能 |
|------|------|
| `scontrol ping` | 檢查控制器狀態 |
| `scontrol show node <name>` | 顯示節點詳情 |
| `scontrol show job <id>` | 顯示作業詳情 |
| `scontrol show slurmd` | 顯示 slurmd 狀態 |
| `sinfo -R` | 顯示節點 DOWN 原因 |
| `squeue -t PD` | 顯示待處理作業 |
| `sdiag` | 顯示排程診斷 |

### 狀態修復指令

| 指令 | 功能 |
|------|------|
| `scontrol update NodeName=X State=resume` | 恢復節點 |
| `scontrol update NodeName=X State=down Reason="..."` | 設定節點 DOWN |
| `scontrol setdebug debug` | 設定除錯等級 |
| `scontrol reconfigure` | 重新載入配置 |
| `scancel -f <jobid>` | 強制取消作業 |

### 日誌檔案位置

| 日誌 | 預設位置 |
|------|----------|
| slurmctld | `/var/log/slurm/slurmctld.log` |
| slurmd | `/var/log/slurm/slurmd.log` |
| slurmdbd | `/var/log/slurm/slurmdbd.log` |

### 常見 DOWN 原因

| 原因 | 說明 | 解決方案 |
|------|------|----------|
| `Not responding` | 節點無回應 | 檢查網路和 slurmd |
| `Low RealMemory` | 記憶體不足 | 修復節點或調整配置 |
| `Low tmp space` | 暫存空間不足 | 清理暫存目錄 |
| `Kill task failed` | 無法終止任務 | 手動清理或重啟 |
