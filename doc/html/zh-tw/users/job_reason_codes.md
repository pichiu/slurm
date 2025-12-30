# Slurm 作業原因代碼

## TL;DR
- 原因代碼說明待處理作業尚未被排程器啟動的原因
- 常見原因：`Resources`（資源不足）、`Priority`（優先級較低）、`Dependency`（依賴未滿足）
- `AssocGrp*`、`AssocMax*` 表示關聯的群組或最大限制已達到
- `QOSGrp*`、`QOSMax*` 表示 QOS 的群組或最大限制已達到
- 使用 `squeue -j <jobid>` 或 `scontrol show job <jobid>` 查看作業原因

---

## Translation

這些原因代碼可用於識別待處理作業尚未被排程器啟動的原因。作業可能有多個無法啟動的原因，在這種情況下，只會顯示嘗試排程方法遇到的原因。

---

## 常見原因

| 原因代碼 | 說明 |
|----------|------|
| **AssocGrp*** | 作業的關聯已達到某個群組限制 |
| **AssocMax*** | 作業請求的一部分超過關聯的某個最大限制（如 PerJob、PerNode）|
| **BeginTime** | 作業的最早開始時間尚未到達 |
| **Dependency** | 此作業對另一個作業有依賴，尚未滿足 |
| **Max*PerAccount** | 作業請求的一部分超過作業 QOS 的每帳號限制 |
| **Priority** | 存在一個或多個更高優先級的作業 |
| **QOSGrp*** | 作業的 QOS 已達到某個群組限制 |
| **QOSMax*** | 作業請求的一部分超過 QOS 的某個最大限制 |
| **Resources** | 作業請求的資源不可用（例如已被其他作業使用）|

---

## 所有原因代碼

### 一般原因

| 原因代碼 | 說明 |
|----------|------|
| AccountingPolicy | 其他原因不匹配時的備用原因 |
| AccountNotAllowed | 作業的帳號不允許在該分割區中 |
| BadConstraints | 作業的要求無法滿足，通常導致作業被拒絕或保持 |
| BeginTime | 作業的最早開始時間尚未到達 |
| BurstBufferOperation | 作業的突發緩衝區操作失敗 |
| BurstBufferResources | 突發緩衝區資源池中資源不足 |
| BurstBufferStageIn | 突發緩衝區外掛程式正在為作業準備環境 |
| Cleaning | 作業正在重新排隊並清理先前的執行 |
| Constraints | 作業的約束條件目前無法滿足 |
| DeadLine | 此作業無法滿足配置的截止時間 |
| Dependency | 此作業對另一個作業有依賴尚未滿足 |
| DependencyNeverSatisfied | 此作業對另一個作業有永遠無法滿足的依賴 |
| FedJobLock | 作業正在等待聯邦中的叢集同步並發出鎖定 |
| InactiveLimit | 作業達到系統 InactiveLimit |
| InvalidAccount | 作業的帳號無效 |
| InvalidQOS | 作業的 QOS 無效 |
| JobArrayTaskLimit | 作業陣列同時執行任務數的限制已達到 |
| JobHeldAdmin | 作業被系統管理員保持 |
| JobHeldUser | 作業被使用者保持 |
| JobHoldMaxRequeue | 作業已重新排隊足夠次數達到 MAX_BATCH_REQUEUE 限制 |
| JobLaunchFailure | 作業無法啟動（可能是檔案系統問題、無效程式名稱等）|
| Licenses | 作業正在等待授權 |
| NodeDown | 作業需要的節點已停機 |
| NonZeroExitCode | 作業以非零退出代碼終止 |
| None | 作業尚未在當前回填週期中被考慮 |
| OutOfMemory | 作業因記憶體不足錯誤而失敗 |
| PartitionConfig | 作業違反分割區的某些限制時的備用原因 |
| PartitionDown | 作業需要的分割區處於 DOWN 狀態 |
| PartitionInactive | 作業需要的分割區處於非活動狀態 |
| PartitionNodeLimit | 作業需要的節點數超出分割區限制 |
| PartitionTimeLimit | 作業的時間限制超過分割區的時間限制 |
| Priority | 存在更高優先級的作業 |
| Prolog | 作業的 Prolog 程式仍在執行 |
| ReqNodeNotAvail | 作業需要的特定節點目前不可用 |
| Reservation | 作業正在等待其預約變為可用 |
| ReservationDeleted | 作業請求的預約已不在系統上 |
| Resources | 作業請求的資源不可用 |
| SchedDefer | 作業請求立即分配但配置了 defer |
| SystemFailure | Slurm 系統、檔案系統、網路等故障 |
| TimeLimit | 作業耗盡了時間限制 |

### 關聯群組限制 (AssocGrp*)

| 原因代碼 | 說明 |
|----------|------|
| AssocGrpBB | 關聯已達到群組突發緩衝區限制 |
| AssocGrpBBMinutes | 關聯已達到突發緩衝區的群組分鐘數限制 |
| AssocGrpBBRunMinutes | 關聯已達到執行中作業突發緩衝區的群組分鐘數限制 |
| AssocGrpBilling | 關聯已達到群組計費限制 |
| AssocGrpBillingMinutes | 關聯已達到計費值的群組分鐘數限制 |
| AssocGrpBillingRunMinutes | 關聯已達到執行中作業計費值的群組分鐘數限制 |
| AssocGrpCpuLimit | 關聯已達到群組 CPU 限制 |
| AssocGrpCPUMinutesLimit | 關聯已達到 CPU 的群組分鐘數限制 |
| AssocGrpCPURunMinutesLimit | 關聯已達到執行中作業 CPU 的群組分鐘數限制 |
| AssocGrpGRES | 關聯已達到群組 GRES 限制 |
| AssocGrpGRESMinutes | 關聯已達到 GRES 的群組分鐘數限制 |
| AssocGrpGRESRunMinutes | 關聯已達到執行中作業 GRES 的群組分鐘數限制 |
| AssocGrpJobsLimit | 關聯已達到群組作業數限制 |
| AssocGrpMemLimit | 關聯已達到群組記憶體限制 |
| AssocGrpMemMinutes | 關聯已達到記憶體的群組分鐘數限制 |
| AssocGrpMemRunMinutes | 關聯已達到執行中作業記憶體的群組分鐘數限制 |
| AssocGrpNodeLimit | 關聯已達到群組節點限制 |
| AssocGrpNodeMinutes | 關聯已達到節點的群組分鐘數限制 |
| AssocGrpNodeRunMinutes | 關聯已達到執行中作業節點的群組分鐘數限制 |
| AssocGrpSubmitJobsLimit | 關聯已達到可執行或待處理作業的群組數量限制 |
| AssocGrpWallLimit | 關聯已達到執行中作業牆鐘時間的群組限制 |

### 關聯最大限制 (AssocMax*)

| 原因代碼 | 說明 |
|----------|------|
| AssocMaxBBPerJob | 突發緩衝區請求超過每作業限制 |
| AssocMaxBBPerNode | 突發緩衝區請求超過每節點限制 |
| AssocMaxCpuPerJobLimit | CPU 請求超過每作業限制 |
| AssocMaxCpuPerNode | CPU 請求超過每節點限制 |
| AssocMaxGRESPerJob | GRES 請求超過每作業限制 |
| AssocMaxGRESPerNode | GRES 請求超過每節點限制 |
| AssocMaxJobsLimit | 使用者允許執行的作業數已達到限制 |
| AssocMaxMemPerJob | 記憶體請求超過每作業限制 |
| AssocMaxMemPerNode | 記憶體請求超過每節點限制 |
| AssocMaxNodePerJobLimit | 節點數請求超過每作業限制 |
| AssocMaxSubmitJobLimit | 使用者允許執行或待處理的作業數已達到限制 |
| AssocMaxWallDurationPerJobLimit | 牆鐘時間請求超過限制 |

### QOS 群組限制 (QOSGrp*)

| 原因代碼 | 說明 |
|----------|------|
| QOSGrpBB | QOS 已達到群組突發緩衝區限制 |
| QOSGrpBilling | QOS 已達到群組計費限制 |
| QOSGrpCpuLimit | QOS 已達到群組 CPU 限制 |
| QOSGrpGRES | QOS 已達到群組 GRES 限制 |
| QOSGrpJobsLimit | QOS 已達到群組作業數限制 |
| QOSGrpMemLimit | QOS 已達到群組記憶體限制 |
| QOSGrpNodeLimit | QOS 已達到群組節點限制 |
| QOSGrpSubmitJobsLimit | QOS 已達到可執行或待處理作業的群組數量限制 |
| QOSGrpWallLimit | QOS 已達到執行中作業牆鐘時間的群組限制 |

### QOS 最大限制 (QOSMax*)

| 原因代碼 | 說明 |
|----------|------|
| QOSMaxBBPerJob | 突發緩衝區請求超過每作業 QOS 限制 |
| QOSMaxBBPerUser | 突發緩衝區請求超過每使用者 QOS 限制 |
| QOSMaxCpuPerJobLimit | CPU 請求超過每作業 QOS 限制 |
| QOSMaxCpuPerUserLimit | CPU 請求超過每使用者 QOS 限制 |
| QOSMaxGRESPerJob | GRES 請求超過每作業 QOS 限制 |
| QOSMaxGRESPerUser | GRES 請求超過每使用者 QOS 限制 |
| QOSMaxJobsPerUserLimit | 使用者允許執行的作業數已達到 QOS 限制 |
| QOSMaxMemoryPerJob | 記憶體請求超過每作業 QOS 限制 |
| QOSMaxMemoryPerUser | 記憶體請求超過每使用者 QOS 限制 |
| QOSMaxNodePerJobLimit | 節點數請求超過每作業 QOS 限制 |
| QOSMaxNodePerUserLimit | 節點數請求超過每使用者 QOS 限制 |
| QOSMaxSubmitJobPerUserLimit | 使用者允許執行或待處理的作業數已達到 QOS 限制 |
| QOSMaxWallDurationPerJobLimit | 牆鐘時間請求超過 QOS 限制 |

### QOS 最小限制 (QOSMin*)

| 原因代碼 | 說明 |
|----------|------|
| QOSMinCpuNotSatisfied | CPU 請求未滿足每作業 QOS 最小要求 |
| QOSMinGRES | GRES 請求未滿足每作業 QOS 最小要求 |
| QOSMinMemory | 記憶體請求未滿足每作業 QOS 最小要求 |
| QOSMinNode | 節點數請求未滿足每作業 QOS 最小要求 |

### 帳號相關限制 (Per Account)

| 原因代碼 | 說明 |
|----------|------|
| MaxBBPerAccount | 突發緩衝區請求超過 QOS 的每帳號限制 |
| MaxCpuPerAccount | CPU 請求超過 QOS 的每帳號限制 |
| MaxGRESPerAccount | GRES 請求超過 QOS 的每帳號限制 |
| MaxJobsPerAccount | 作業數超過 QOS 的每帳號限制 |
| MaxMemoryPerAccount | 記憶體請求超過 QOS 的每帳號限制 |
| MaxNodePerAccount | 節點數請求超過 QOS 的每帳號限制 |
| MaxSubmitJobsPerAccount | 執行或待處理作業數超過 QOS 的每帳號限制 |

---

## Practical Example

### 診斷作業待處理原因

```bash
# 查看作業狀態和原因
$ squeue -j 12345
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
12345     batch  my_job     user PD       0:00      4 (Resources)

# 查看詳細原因
$ scontrol show job 12345 | grep -E "State|Reason"
   JobState=PENDING Reason=Resources Dependency=(null)

# 常見原因及解決方法：

# 1. Resources - 資源不足
# 解決：等待資源釋放，或減少資源請求
$ squeue -j 12345 -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"

# 2. Priority - 優先級較低
# 解決：等待更高優先級作業完成
$ sprio -j 12345

# 3. QOSMaxCpuPerUserLimit - 超過 QOS CPU 限制
# 解決：等待其他作業完成，或減少 CPU 請求
$ sacctmgr show qos format=name,maxcpusperuser

# 4. AssocGrpCpuLimit - 關聯群組 CPU 限制
# 解決：等待同帳號其他作業完成
$ sacctmgr show assoc user=$USER format=user,account,grptres
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **混淆 Resources 和 Priority**
   - `Resources`：叢集資源不足
   - `Priority`：有更高優先級的作業在等待相同資源

2. **忽略限制類型**
   - `Grp*`：群組限制（同帳號/QOS 所有作業的總和）
   - `Max*PerJob`：單一作業的限制
   - `Max*PerUser`：單一使用者所有作業的總和

3. **不了解 None 原因**
   - `None` 表示作業尚未在當前回填週期中被評估
   - 這是暫時狀態，通常會很快更新

### 實用建議

- **查看詳細資訊**：`scontrol show job` 比 `squeue` 提供更多細節
- **檢查限制**：使用 `sacctmgr show qos` 和 `sacctmgr show assoc` 查看限制
- **監控資源**：`sinfo` 顯示可用資源
- **調整請求**：如果經常遇到限制，考慮減少資源請求或分割作業

---

## Quick Reference

### 最常見原因

| 原因 | 含義 | 解決方法 |
|------|------|----------|
| Resources | 資源不足 | 等待或減少請求 |
| Priority | 優先級較低 | 等待 |
| Dependency | 依賴未滿足 | 等待依賴作業完成 |
| QOSMaxCpuPerUserLimit | 超過使用者 CPU 限制 | 等待其他作業完成 |
| AssocGrpCpuLimit | 超過群組 CPU 限制 | 等待同帳號作業完成 |
| PartitionTimeLimit | 超過分割區時間限制 | 減少時間請求 |
| JobHeldUser | 使用者保持 | `scontrol release <jobid>` |
| ReqNodeNotAvail | 請求的節點不可用 | 檢查節點狀態或移除特定節點請求 |

### 查看原因的指令

```bash
squeue -j <jobid>                    # 簡短原因
scontrol show job <jobid>            # 詳細資訊
squeue -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"  # 自訂格式
```
