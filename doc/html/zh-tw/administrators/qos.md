# Slurm 服務品質 (QOS)

## TL;DR
- **QOS (Quality of Service)** 可定義作業的優先級、搶占權限和資源限制
- 使用 `sacctmgr` 建立和管理 QOS，提交作業時用 `--qos=<名稱>` 指定
- QOS 限制會覆蓋使用者/帳號/叢集/分割區關聯的限制
- 可設定 QOS 搶占其他 QOS 的作業（需 PreemptType=preempt/qos）
- 分割區 QOS 和作業 QOS 是不同的概念，分割區 QOS 優先級較高

---

## Translation

### 概述

可以為提交到 Slurm 的每個作業指定服務品質 (Quality of Service, QOS)。QOS 使用 *sacctmgr* 指令在 Slurm 資料庫中定義。作業使用 *sbatch*、*salloc* 和 *srun* 指令的「--qos=」選項請求 QOS。

### 目錄
- [對作業的影響](#對作業的影響)
  - [排程優先級](#排程優先級)
  - [搶占](#搶占)
  - [資源限制](#資源限制)
- [分割區 QOS](#分割區-qos)
- [相對 QOS](#相對-qos)
- [其他 QOS 選項](#其他-qos-選項)
- [配置](#配置)
- [範例](#範例)

### 對作業的影響

與作業關聯的 QOS 將以三種關鍵方式影響作業：排程優先級、搶占和資源限制。

#### 排程優先級

作業排程優先級由多個因素組成，如 [priority/multifactor](priority_multifactor.html) 外掛程式中所述。其中一個因素是 QOS 優先級。每個 QOS 在 Slurm 資料庫中定義，並包含一個關聯的優先級。請求並被允許使用 QOS 的作業將在作業的[多因素優先級計算](priority_multifactor.html#general)中納入與該 QOS 關聯的優先級。

要啟用多因素優先級計算中的 QOS 優先級組件，必須在 slurm.conf 檔案中定義「PriorityWeightQOS」配置參數並賦予大於零的整數值。

作業的 QOS 僅在載入多因素外掛程式時影響其排程優先級。

#### 搶占

Slurm 提供兩種方式讓排隊中的作業搶占執行中的作業、釋放執行中作業的資源並將其分配給排隊中的作業。詳見[搶占說明](preempt.html)。

搶占方法由 slurm.conf 中定義的「PreemptType」配置參數決定。當「PreemptType」設定為「preempt/qos」時，排隊中作業的 QOS 將用於確定它是否可以搶占執行中的作業。需要注意的是，用於確定作業是否有資格被搶占的 QOS 是與作業關聯的 QOS，而不是[分割區 QOS](#分割區-qos)。

QOS 可以（使用 *sacctmgr*）被分配一個它可以搶占的其他 QOS 列表。當有一個 QOS 允許搶占另一個 QOS 的執行中作業的排隊作業時，Slurm 排程器將搶占執行中的作業。

QOS 選項 PreemptExemptTime 指定作業被考慮搶占前的最短執行時間。QOS 選項優先於同名的全域選項。具有 PreemptExemptTime 的分割區 QOS 優先於具有 PreemptExemptTime 的作業 QOS，除非作業 QOS 啟用了 OverPartQOS 旗標。

#### 資源限制

每個 QOS 被分配一組將應用於作業的限制。這些限制與 Slurm 資料庫中定義的使用者/帳號/叢集/分割區關聯所施加的限制相同，並在[資源限制頁面](resource_limits.html)中描述。當為 QOS 定義了限制時，它們將優先於關聯的限制。

### 分割區 QOS

QOS 可以附加到分割區。這表示分割區將具有與 QOS 相同的所有限制。這不會將作業與 QOS 關聯，也不會給作業任何已分配 QOS 的優先級或搶占特性。作業可以單獨請求相同的 QOS 或不同的 QOS 以獲得這些特性。但是，分割區 QOS 限制將覆蓋作業的 QOS。如果需要相反的效果，您可以使用 `Flags=OverPartQOS` 配置作業的 QOS，這將反轉優先順序。

此功能可用於實作真正的「浮動」分割區，其中分割區可以存取有限數量的資源，而不限制使用哪些節點來獲取資源。這是透過將所有節點分配給分割區，然後配置具有設定為所需資源限制的 `GrpTRES` 的分割區 QOS 來完成的。

**注意**：大多數 QOS 屬性是使用 **sacctmgr** 指令設定的。但是，將 QOS 設定為分割區 QOS 是透過在關聯分割區配置中的 [QOS=](slurm.conf.html#OPT_QOS) 選項在 **slurm.conf** 中完成的。QOS 應在被分配為分割區 QOS 之前使用 **sacctmgr** 建立。可以刪除附加到分割區的 QOS 而不刪除 **slurm.conf** 中的附加。這是不建議的，可能會導致意外行為。

### 相對 QOS

從 Slurm 23.11 開始，可以透過設定 `Flags=Relative` 將 QOS 配置為包含相對資源限制而不是絕對限制。當設定此旗標時，所有資源限制都被視為可用總資源的百分比。高於 100 的值被解釋為 100%。記憶體限制應設定為無單位。雖然會顯示預設單位（MB），但限制將作為百分比強制執行（1MB = 1%）。

**注意**：當將 *Flags=Relative* 添加到 QOS 時，必須重新啟動或重新配置 **slurmctld** 才能使旗標生效。

通常，相對 QOS 上的限制將相對於整個叢集中的資源計算。例如，`cpu=50` 將被解釋為叢集中所有 CPU 的 50%。

但是，當相對 QOS 也被分配為分割區 QOS 時，將適用一些獨特的條件：

1. 限制將相對於分割區的資源計算；例如，`cpu=50` 將被解釋為關聯分割區中所有 CPU 的 50%。
2. 只有一個分割區可以將此 QOS 作為其分割區 QOS。
3. 作業將不被允許將其作為普通 QOS 使用。
   **注意**：為避免意外的作業提交錯誤，建議不要將相對分割區 QOS 添加到任何基於關聯的實體。

### 其他 QOS 選項

**Flags** - slurmctld 用於覆蓋或強制執行某些特性。要清除先前設定的值，請使用新值 -1 的修改指令。有效選項包括：

- **DenyOnLimit**：如果設定，使用此 QOS 的作業如果作為獨立作業不符合 QOS「Max」限制，將在提交時被拒絕。當考慮其他作業時超過這些限制但單獨考慮時符合限制的作業不會被拒絕。相反，它們將待處理直到資源可用（如沒有 DenyOnLimit 的預設值）。群組限制（例如 **GrpTRES**）也將被視為「Max」限制（例如 **MaxTRESPerNode**），如果作業作為獨立作業會違反限制，則會被拒絕。這目前僅適用於 QOS 和關聯限制。

- **EnforceUsageThreshold**：如果設定，且 QOS 也有 UsageThreshold，任何使用此 QOS 提交的低於 UsageThreshold 的作業將被保留，直到其公平份額使用量超過閾值。

- **NoDecay**：如果設定，此 QOS 不會讓其 GrpTRESMins、GrpWall 和 UsageRaw 被 slurm.conf 的 PriorityDecayHalfLife 或 PriorityUsageResetPeriod 設定衰減。這允許 QOS 提供聚合限制，一旦消耗，將不會自動補充。這樣的 QOS 將作為可存取它的關聯的資源的有時限配額。使用 QOS 的關聯的帳號/使用者使用量仍將衰減。QOS GrpTRESMins 和 GrpWall 限制可以增加，或者 QOS RawUsage 值可以重設為 0（零），以再次允許使用此 QOS 提交的作業執行（如果因 QOSGrp{TRES}MinutesLimit 或 QOSGrpWallLimit 原因待處理，其中 {TRES} 是某種類型的可追蹤資源）。

- **NoReserve**：如果設定此旗標且使用回填排程，使用此 QOS 的作業將不會在回填排程的隨時間分配資源的地圖中保留資源。此旗標旨在與可被所有其他 QOS 關聯的作業搶占的 QOS 一起使用（例如與「standby」QOS 一起使用）。如果此旗標與不能被所有其他 QOS 搶占的 QOS 一起使用，可能會導致較大作業的飢餓。

- **OverPartQOS**：如果設定，使用此 QOS 的作業將能夠覆蓋請求分割區的 QOS 限制使用的任何限制。

- **PartitionMaxNodes**：如果設定，使用此 QOS 的作業將能夠覆蓋請求分割區的 MaxNodes 限制。

- **PartitionMinNodes**：如果設定，使用此 QOS 的作業將能夠覆蓋請求分割區的 MinNodes 限制。

- **PartitionTimeLimit**：如果設定，使用此 QOS 的作業將能夠覆蓋請求分割區的 TimeLimit。

- **Relative**：如果設定，QOS 限制將被視為叢集或分割區的百分比而不是絕對限制（數字應小於 100）。在將 *Relative* 旗標添加到 QOS 後，應重新啟動或重新配置控制器。如果這用作分割區 QOS：
  1. 限制將相對於分割區的資源計算。
  2. 只有一個分割區可以將此 QOS 作為其分割區 QOS。
  3. 作業將不被允許將其作為普通 QOS 使用。

- **RequiresReservation**：如果設定，使用此 QOS 的作業在提交作業時必須指定預約。此選項可用於限制可能具有更大搶占能力或額外資源的 QOS 的使用，使其只允許在預約內使用。

- **UsageFactorSafe**：如果設定，且 **AccountingStorageEnforce** 包含 **Safe**，作業只有在應用 **UsageFactor** 後作業可以執行到完成時才能執行。

**GraceTime** - 延長給已被選中搶占的作業的搶占寬限時間。

**UsageFactor** - 一個會計入作業 TRES 使用量（例如 RawUsage、TRESMins、TRESRunMins）的浮點數。例如，如果使用因子是 2，作業每執行一個 TRESBillingUnit 秒將計為 2。如果使用因子是 .5，每秒只計為一半的時間。設定為 0 將不會從作業添加任何計時使用量。

使用因子僅適用於作業的 QOS，而不適用於分割區 QOS。

如果設定了 **UsageFactorSafe** 旗標**且** **AccountingStorageEnforce** 包含 **Safe**，作業只有在應用 **UsageFactor** 後作業可以執行到完成時才能執行。

如果**未**設定 **UsageFactorSafe** 旗標且 **AccountingStorageEnforce** 包含 **Safe**，作業將能夠在未應用 **UsageFactor** 的情況下被排程，並能夠執行而不會因限制被終止。

如果**未**設定 **UsageFactorSafe** 旗標且 **AccountingStorageEnforce** 不包含 **Safe**，作業將能夠在未應用 **UsageFactor** 的情況下被排程，並可能因限制被終止。

詳見 slurm.conf man 頁面中的 **AccountingStorageEnforce**。

預設值為 1。要清除先前設定的值，請使用新值 -1 的修改指令。

**UsageThreshold** - 表示允許執行作業的關聯的最低公平份額的浮點數。如果關聯低於此閾值並有待處理作業或提交新作業，這些作業將被保留直到使用量回到閾值以上。使用 *sshare* 查看系統上當前的份額。

### 配置

總結以上內容，QOS 及其關聯的限制使用 *sacctmgr* 工具在 Slurm 資料庫中定義。只有在載入多因素優先級外掛程式並在 slurm.conf 檔案中定義非零的「PriorityWeightQOS」時，QOS 才會影響作業排程優先級。只有當「PreemptType」在 slurm.conf 檔案中定義為「preempt/qos」時，QOS 才會決定作業搶占。為 QOS 定義的限制（如上所述）將覆蓋使用者/帳號/叢集/分割區關聯的限制。

---

## Explanation

### QOS 與關聯限制的優先順序

```
最高優先級 ──────────────────────────── 最低優先級
分割區 QOS > 作業 QOS > 帳號關聯 > 使用者關聯
                ↑
        (除非設定 OverPartQOS)
```

### 常用 QOS 參數

| 參數 | 說明 |
|------|------|
| Priority | QOS 優先級權重 |
| GrpTRES | 此 QOS 所有作業可使用的總 TRES |
| GrpTRESMins | 此 QOS 所有作業可使用的 TRES 分鐘數 |
| MaxTRESPerUser | 每個使用者可使用的最大 TRES |
| MaxJobsPerUser | 每個使用者同時執行的最大作業數 |
| MaxSubmitJobsPerUser | 每個使用者可提交的最大作業數 |
| Preempt | 此 QOS 可搶占的其他 QOS 列表 |
| GraceTime | 搶占寬限時間（秒）|
| UsageFactor | 使用量計算係數 |

---

## Practical Example

### 場景：為不同優先級使用者配置 QOS

研究中心需要為不同類型的使用者設定不同的資源配額和優先級。

```bash
# 1. 查看目前的 QOS
$ sacctmgr show qos format=name,priority
      Name   Priority
---------- ----------
    normal          0

# 2. 建立高優先級 QOS
$ sacctmgr add qos high_priority
# add qos        建立新的 QOS
# high_priority  QOS 名稱

# 3. 設定 QOS 優先級和限制
$ sacctmgr modify qos high_priority set \
    priority=100 \
    GrpTRES=cpu=200,mem=800G \
    MaxTRESPerUser=cpu=50,mem=200G \
    MaxJobsPerUser=10
# priority=100              優先級權重為 100
# GrpTRES=cpu=200,mem=800G  此 QOS 總共可用 200 CPU、800G 記憶體
# MaxTRESPerUser=...        每使用者最多 50 CPU、200G 記憶體
# MaxJobsPerUser=10         每使用者最多同時執行 10 個作業

# 4. 建立低優先級（可被搶占）的 QOS
$ sacctmgr add qos low_priority
$ sacctmgr modify qos low_priority set \
    priority=10 \
    GrpTRES=cpu=100 \
    MaxTRESPerUser=cpu=20 \
    MaxJobsPerUser=5

# 5. 設定搶占關係：high_priority 可搶占 low_priority
$ sacctmgr modify qos high_priority set preempt=low_priority

# 6. 將 QOS 分配給使用者
$ sacctmgr modify user professor set qos=high_priority
$ sacctmgr modify user student set qos=low_priority

# 7. 允許使用者使用多個 QOS
$ sacctmgr modify user professor set qos+=low_priority,normal
# +=  表示添加而非取代

# 8. 查看使用者的 QOS 設定
$ sacctmgr show assoc format=cluster,user,qos
   Cluster       User                  QOS
---------- ---------- --------------------
  myslurm  professor  high_priority,low_priority,normal
  myslurm    student              low_priority

# 9. 提交作業時指定 QOS
$ sbatch --qos=high_priority my_job.sh
```

### 配置分割區 QOS（浮動分割區）

```bash
# 建立用於分割區的 QOS
$ sacctmgr add qos floating_qos
$ sacctmgr modify qos floating_qos set \
    GrpTRES=cpu=100,mem=400G

# 在 slurm.conf 中設定分割區 QOS
# PartitionName=floating Nodes=ALL QOS=floating_qos

# 這樣 floating 分割區最多只能使用 100 CPU 和 400G 記憶體，
# 但可以從任何節點取得這些資源
```

### 配置相對限制 QOS

```bash
# 建立相對限制的 QOS（23.11+）
$ sacctmgr add qos relative_qos
$ sacctmgr modify qos relative_qos set \
    Flags=Relative \
    GrpTRES=cpu=50,mem=50
# cpu=50 表示叢集 50% 的 CPU
# mem=50 表示叢集 50% 的記憶體

# 重新配置 slurmctld 使旗標生效
$ scontrol reconfigure
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **混淆分割區 QOS 和作業 QOS**
   ```bash
   # 誤解：以為分割區 QOS 會給作業搶占能力
   # 分割區 QOS 只提供限制，不提供優先級或搶占特性

   # 正確理解：
   # - 分割區 QOS：限制分割區的總資源使用
   # - 作業 QOS：影響作業的優先級、搶占和限制
   ```

2. **忘記啟用 PriorityWeightQOS**
   ```bash
   # 問題：設定了 QOS 優先級但不生效

   # 解決：在 slurm.conf 添加
   PriorityWeightQOS=1000
   ```

3. **刪除仍在使用的 QOS**
   ```bash
   # 錯誤：刪除附加到分割區的 QOS
   $ sacctmgr delete qos partition_qos
   # 可能導致意外行為

   # 正確：先從 slurm.conf 移除 QOS=，然後重新配置，最後刪除
   ```

4. **相對 QOS 配置錯誤**
   ```bash
   # 錯誤：設定 Relative 旗標後沒有重新配置
   $ sacctmgr modify qos myqos set Flags=Relative
   # 不生效

   # 正確：必須重新配置或重啟 slurmctld
   $ scontrol reconfigure
   ```

### 實用建議

- **使用 format 選項查看 QOS**：
  ```bash
  # 預設輸出很長，用 format 過濾
  $ sacctmgr show qos format=name,priority,grptres,preempt,flags
  ```

- **謹慎使用 NoDecay**：資源配額一旦用完不會自動恢復，適合時限配額場景

- **設定合理的 GraceTime**：給被搶占的作業一些時間清理
  ```bash
  $ sacctmgr modify qos low_priority set gracetime=60
  ```

- **監控 QOS 使用情況**：
  ```bash
  # 查看 QOS 使用狀態
  $ sshare -l

  # 查看特定 QOS 的作業
  $ squeue --qos=high_priority
  ```

---

## Quick Reference

### sacctmgr QOS 指令

| 指令 | 說明 |
|------|------|
| `sacctmgr add qos <name>` | 建立新 QOS |
| `sacctmgr show qos` | 顯示所有 QOS |
| `sacctmgr modify qos <name> set ...` | 修改 QOS |
| `sacctmgr delete qos <name>` | 刪除 QOS |

### 常用 QOS 屬性

| 屬性 | 說明 |
|------|------|
| `Priority=<int>` | 優先級權重 |
| `GrpTRES=<tres>` | 群組 TRES 限制 |
| `MaxTRESPerUser=<tres>` | 每使用者最大 TRES |
| `MaxJobsPerUser=<int>` | 每使用者最大執行作業數 |
| `MaxSubmitJobsPerUser=<int>` | 每使用者最大提交作業數 |
| `Preempt=<qos_list>` | 可搶占的 QOS 列表 |
| `GraceTime=<seconds>` | 搶占寬限時間 |
| `UsageFactor=<float>` | 使用量係數 |
| `Flags=<flag_list>` | 旗標設定 |

### 使用者 QOS 管理

| 指令 | 說明 |
|------|------|
| `sacctmgr modify user <name> set qos=<qos>` | 設定使用者 QOS |
| `sacctmgr modify user <name> set qos+=<qos>` | 添加使用者 QOS |
| `sacctmgr show assoc format=user,qos` | 查看使用者 QOS |

### 作業提交

```bash
sbatch --qos=<qos_name> job.sh    # 指定 QOS 提交作業
```
