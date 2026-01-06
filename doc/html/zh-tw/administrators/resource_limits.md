# Slurm 資源限制

## TL;DR
- 資源限制按階層強制執行：分割區 QOS > 作業 QOS > 使用者關聯 > 帳號關聯 > 叢集 > 分割區
- 使用 `AccountingStorageEnforce` 啟用限制強制執行
- 限制類型：GrpTRES（群組總量）、MaxTRESPerUser（每使用者最大值）、MaxJobs（最大作業數）等
- 使用 `sacctmgr` 管理關聯和 QOS 的限制
- 限制變更會立即生效，但不會取消已執行的作業

---

## Translation

### 概述

強烈建議在使用本文件之前熟悉 Slurm 的[會計](accounting.md)網頁。

### 階層

Slurm 的階層限制按以下順序強制執行，其中作業 QOS 和分割區 QOS 的順序可透過使用 QOS 旗標「OverPartQOS」來反轉：

1. 分割區 QOS 限制
2. 作業 QOS 限制
3. 使用者關聯
4. 帳號關聯（向上遞升階層）
5. 根/叢集關聯
6. 分割區限制
7. 無

**注意**：如果在此階層的多個點定義了限制，將使用此列表中首先定義限制的點。考慮以下範例：
- 分割區 QOS 中 MaxJobs=20 且 MaxSubmitJobs 未定義
- 作業 QOS 中沒有設定限制
- 使用者關聯中 MaxJobs=4 且 MaxSubmitJobs=50

生效的限制將是 MaxJobs=20 和 MaxSubmitJobs=50。

**注意**：除了以下限制外，上述指定的優先順序都會被遵守：Max[Time|Wall]、[Min|Max]Nodes。對於這些限制，即使作業被 QOS 和/或關聯限制強制執行，它也不能超過分割區級別施加的限制，即使它在列表底部。因此這 3 種限制的預設值是它們以分割區為上限。如果設定了相應的 QOS PartitionTimeLimit 和/或 Partition[Max|Min]Nodes 旗標，則可以忽略此分割區級別上限，然後作業將按照上述順序強制執行 QOS 和/或關聯級別施加的限制。

**Grp*** 限制也是例外。帳號級別的更嚴格限制將在使用者級別的較寬鬆限制之前被強制執行。這是因為正在強制執行的限制的性質要求不超過最高級別的限制。

### 配置

排程策略資訊必須儲存在由 **slurm.conf** 配置檔案中的 **AccountingStorageType** 配置參數指定的資料庫中。資訊可以記錄在 [MySQL](http://www.mysql.com/) 或 [MariaDB](https://mariadb.org/) 資料庫中。出於安全和效能原因，強烈建議使用 SlurmDBD（Slurm 資料庫守護程序）作為資料庫的前端。SlurmDBD 使用 Slurm 認證外掛程式（例如 MUNGE）。SlurmDBD 還使用現有的 Slurm 會計儲存外掛程式以最大化程式碼重用。SlurmDBD 使用資料快取和待處理請求的優先順序排序以優化效能。雖然 SlurmDBD 依賴現有的 Slurm 外掛程式進行認證和資料庫使用，但安裝 SlurmDBD 的主機上不需要其他 Slurm 指令和守護程序。SlurmDBD 執行只需要 *slurmdbd* 和 *slurm-plugins* RPM。

會計和排程策略都是基於*關聯* (association) 配置的。*關聯*是由叢集名稱、帳號、使用者和（可選的）Slurm 分割區組成的 4 元組。要強制執行排程策略，請設定 **AccountingStorageEnforce** 的值。此選項包含您可能想要強制執行的選項的逗號分隔列表。有效選項包括：

- **associations**：如果使用者的*關聯*不在資料庫中，這將防止使用者執行作業。此選項將防止使用者存取無效的帳號。

- **limits**：這將強制執行設定給關聯的限制。透過設定此選項，「associations」選項也會被設定。

- **qos**：這將要求所有作業指定（明確或預設）有效的 qos（服務品質）。QOS 值在資料庫中為每個關聯定義。透過設定此選項，「associations」選項也會被設定。

- **safe**：這將確保只有在使用具有 TRES 分鐘限制的關聯或 qos 時，如果作業能夠執行到完成，作業才會被啟動。如果沒有設定此選項，只要使用量還沒有達到 TRES 分鐘限制，作業就會被啟動，這可能導致作業被啟動但在達到限制時被終止。設定「safe」選項後，作業不會因限制而被終止，即使限制在作業啟動後被更改且關聯或 qos 違反了更新的限制。透過設定此選項，「associations」選項和「limits」選項都會自動設定。

- **wckeys**：這將防止使用者在他們無權存取的 wckey 下執行作業。透過使用此選項，「associations」選項也會被設定。「TrackWCKey」選項也會設定為 true。

**注意**：關聯是叢集、帳號、使用者名稱和可選分割區名稱的組合。如果沒有設定 AccountingStorageEnforce（預設行為），作業將根據每個叢集上 Slurm 配置的策略執行。

### 工具

用於管理會計策略的工具是 *sacctmgr*。它可用於建立和刪除叢集、使用者、帳號和分割區記錄以及它們的組合*關聯*記錄。請參閱 *man sacctmgr* 以獲取此工具的詳細資訊和使用範例。

對排程策略所做的更改會上傳到各個叢集上的 Slurm 控制守護程序並立即生效。當關聯被刪除時，屬於該關聯的所有執行中或待處理作業會立即被取消。當降低限制時，執行中的作業不會被取消以滿足新限制，但新的較低限制將被強制執行。

### 關聯特定限制和排程策略

這些代表與關聯相關的限制和排程策略。在處理關聯時，這些限制大多不僅可用於使用者關聯，還可用於每個叢集和帳號。限制和策略按以下順序應用：

1. 為使用者關聯指定的選項。
2. 為帳號指定的選項。
3. 為叢集指定的選項。
4. 如果上述級別沒有配置任何內容，則不會應用任何限制。

這些只是關聯的限制和策略。有關可顯示欄位的更完整描述，請參閱 [sacctmgr](sacctmgr.html#SECTION_LIST/SHOW-ASSOCIATION-FORMAT-OPTIONS) man 頁面。

**Fairshare** - 用於確定優先級的整數值。本質上這是此關聯及其子關聯對上述系統的要求量。也可以是字串「parent」，當用於使用者時這表示父關聯用於公平份額。如果在帳號上設定 Fairshare=parent，該帳號的子關聯在公平份額計算中將有效地重新指定父級為其父級的第一個不是 Fairshare=parent 的父級。限制保持不變，只有其公平份額值受到影響。

**GrpJobs** - 從一個關聯及其子關聯在任何給定時間能夠執行的作業總數。如果達到此限制，新作業將排隊，但只有在此群組的先前作業完成後才允許執行。

**GrpJobsAccrue** - 從一個關聯及其子關聯在任何給定時間能夠累積年齡優先級的待處理作業總數。如果達到此限制，新作業將排隊但不累積年齡優先級，直到此群組的先前作業從待處理中移除。此限制不決定作業是否可以執行，它只限制優先級的年齡因素。

**GrpSubmitJobs** - 從一個關聯及其子關聯在任何給定時間能夠提交到系統的作業總數。如果達到此限制，新提交請求將被拒絕，直到此群組的先前作業完成。

**GrpTRES** - 從一個關聯及其子關聯執行的作業在任何給定時間能夠使用的 TRES 總數。如果達到此限制，新作業將排隊，但只有在此群組的資源被釋放後才允許執行。

**GrpTRESMins** - 從一個關聯及其子關聯執行的過去、現在和未來作業可能使用的 TRES 分鐘總數。如果達到任何限制，此群組中所有使用該 TRES 的執行中作業將被終止，且不允許執行新作業。此使用量會衰減（以 PriorityDecayHalfLife 的速率）。它也可以被重設（根據 PriorityUsageResetPeriod）以允許作業再次針對關聯樹執行。此限制僅在使用優先級多因素外掛程式時適用。

**GrpTRESRunMins** - 用於限制與一個關聯及其子關聯一起執行的所有作業使用的 TRES 分鐘的組合總數。這會考慮執行中作業的時間限制並消耗它。如果達到限制，在其他作業完成以釋放時間之前，不會啟動新作業。

**GrpWall** - 為一個關聯及其子關聯聚合分配的執行中作業的最大牆鐘時間。如果達到此限制，此關聯中的未來作業將排隊，直到它們能夠在限制內執行。此使用量會衰減（以 PriorityDecayHalfLife 的速率）。它也可以被重設（根據 PriorityUsageResetPeriod）以允許作業再次針對關聯樹執行。

**MaxJobs** - 給定關聯在任何給定時間能夠執行的作業總數。如果達到此限制，新作業將排隊，但只有在關聯中的現有作業完成後才允許執行。

**MaxJobsAccrue** - 給定關聯在任何給定時間能夠累積年齡優先級的待處理作業的最大數量。如果達到此限制，新作業將排隊但不會累積年齡優先級，直到關聯中的現有作業從待處理狀態移動。此限制不決定作業是否可以執行，它只限制優先級的年齡因素。

**MaxSubmitJobs** - 從給定關聯在任何給定時間能夠提交到系統的作業的最大數量。如果達到此限制，新提交請求將被拒絕，直到此關聯中的現有作業完成。

**MaxTRESMinsPerJob** - 作業使用的 TRES 分鐘限制。如果達到此限制，如果不在安全模式下執行，作業將被終止，否則作業將待處理直到有足夠的時間來完成作業。

**MaxTRESPerJob** - 從關聯中任何給定作業可以擁有的最大 TRES 大小。

**MaxTRESPerNode** - 作業分配中每個節點可以使用的最大 TRES 大小。

**MaxWallDurationPerJob** - 給定關聯中任何個別作業可以執行的最大牆鐘時間。如果達到此限制，作業將在提交時被拒絕。

**MinPrioThreshold** - 在給定關聯中保留資源所需的最低優先級。用於覆蓋 bf_min_prio_reserve。詳見 [bf_min_prio_reserve](slurm.conf.html#OPT_bf_min_prio_reserve=#)。

**QOS** - 關聯能夠執行的 QOS 的逗號分隔列表。

**注意**：使用 *sacctmgr* 修改 TRES 欄位時，必須指定要修改的 TRES（完整列表請參見 [TRES](tres.md)），如以下範例所示：

```bash
# 設定：
sacctmgr modify user bob set GrpTRES=cpu=1500,mem=200,gres/gpu=50

# 取消設定：
sacctmgr modify user bob set GrpTRES=cpu=-1,mem=-1,gres/gpu=-1
```

### QOS 特定限制和排程策略

如[上述](#階層)所述，預設行為是設定在分割區 QOS 上的限制將在作業請求的 QOS 上的限制之前應用。您可以使用 *OverPartQOS* 旗標更改此行為。

除非另有說明，如果作業請求本身違反了給定限制，作業將待處理，除非作業的 QOS 設定了 DenyOnLimit 旗標，這將導致作業在提交時被拒絕。當考慮 Grp 限制與此旗標相關時，Grp 限制被視為 Max 限制。

**GraceTime** - 延長給已被選中搶占的作業的搶占寬限時間，格式為 &lt;hh&gt;:&lt;mm&gt;:&lt;ss&gt;。預設值為零，表示此 QOS 不允許搶占寬限時間。此值僅對 QOS PreemptMode=CANCEL 和 PreemptMode=REQUEUE 有意義。

**GrpJobs** - 從一個 QOS 在任何給定時間能夠執行的作業總數。如果達到此限制，新作業將排隊，但只有在此群組的先前作業完成後才允許執行。

**GrpJobsAccrue** - 從一個 QOS 在任何給定時間能夠累積年齡優先級的待處理作業總數。如果達到此限制，新作業將排隊但不會累積基於年齡的優先級，直到此群組的先前作業從待處理中移除。此限制不決定作業是否可以執行，它只限制優先級的年齡因素。此限制僅適用於作業的 QOS，而不適用於分割區的 QOS。

**GrpSubmitJobs** - 從一個 QOS 在任何給定時間能夠提交到系統的作業總數。如果達到此限制，新提交請求將被拒絕，直到此群組的先前作業完成。

**GrpTRES** - 從一個 QOS 執行的作業在任何給定時間能夠使用的 TRES 總數。如果達到此限制，新作業將排隊，但只有在此群組的資源被釋放後才允許執行。

**GrpTRESMins** - 從一個 QOS 執行的過去、現在和未來作業可能使用的 TRES 分鐘總數。如果達到任何限制，此群組中所有使用該 TRES 的執行中作業將被終止，且不允許執行新作業。此使用量會衰減（以 PriorityDecayHalfLife 的速率）。它也可以被重設（根據 PriorityUsageResetPeriod）以允許作業再次針對 QOS 執行。設定了 NoDecay 旗標的 QOS 不會衰減 GrpTRESMins，詳見 [QOS 選項](qos.md#qos_other)。此限制僅在使用優先級多因素外掛程式時適用。

**GrpTRESRunMins** - 用於限制與一個 QOS 一起執行的所有作業使用的 TRES 分鐘的組合總數。這會考慮執行中作業的時間限制並消耗它。如果達到限制，在其他作業完成以釋放時間之前，不會啟動新作業。

**GrpWall** - 為一個 QOS 聚合分配的執行中作業的最大牆鐘時間。如果達到此限制，此 QOS 中的未來作業將排隊，直到它們能夠在限制內執行。此使用量會衰減（以 PriorityDecayHalfLife 的速率）。它也可以被重設（根據 PriorityUsageResetPeriod）以允許作業再次針對 QOS 執行。設定了 NoDecay 旗標的 QOS 不會衰減 GrpWall。詳見 [QOS 選項](qos.md#qos_other)。

**LimitFactor** - 一個會計入關聯的 [Grp|Max]TRES 限制的浮點數。例如，如果 LimitFactor 是 2，則 GrpTRES 為 30 CPU 的關聯在此 QOS 下執行時將被允許分配 60 CPU。**注意**：此因子僅適用於在此 QOS 中執行的關聯，不適用於 QOS 本身的任何限制。

**MaxJobsAccruePerAccount** - 一個帳號（或子帳號）在任何給定時間可以累積年齡優先級的待處理作業的最大數量。此限制不決定作業是否可以執行，它只限制優先級的年齡因素。

**MaxJobsAccruePerUser** - 一個使用者在任何給定時間可以累積年齡優先級的待處理作業的最大數量。此限制不決定作業是否可以執行，它只限制優先級的年齡因素。

**MaxJobsPerAccount** - 一個帳號（或子帳號）在給定時間可以執行的作業的最大數量。

**MaxJobsPerUser** - 一個使用者在給定時間可以執行的作業的最大數量。

**MaxSubmitJobsPerAccount** - 一個帳號（或子帳號）在給定時間可以執行和待處理的作業的最大數量。

**MaxSubmitJobsPerUser** - 一個使用者在給定時間可以執行和待處理的作業的最大數量。

**MaxTRESMinsPerJob** - 每個作業能夠使用的最大 TRES 分鐘數。

**MaxTRESPerAccount** - 一個帳號在給定時間可以分配的最大 TRES 數。

**MaxTRESPerJob** - 每個作業能夠使用的最大 TRES 數。

**MaxTRESPerNode** - 作業分配中每個節點可以使用的最大 TRES 數。

**MaxTRESPerUser** - 一個使用者在給定時間可以分配的最大 TRES 數。

**MaxWallDurationPerJob** - 每個作業能夠使用的最大牆鐘時間。格式為 &lt;min&gt; 或 &lt;min&gt;:&lt;sec&gt; 或 &lt;hr&gt;:&lt;min&gt;:&lt;sec&gt; 或 &lt;days&gt;-&lt;hr&gt;:&lt;min&gt;:&lt;sec&gt; 或 &lt;days&gt;-&lt;hr&gt;。該值以分鐘記錄，並根據需要四捨五入。

**MinPrioThreshold** - 排程時保留資源所需的最低優先級。

**MinTRESPerJob** - 使用請求的 QOS 時任何給定作業可以擁有的最小 TRES 大小。

**UsageFactor** - 一個會計入作業 TRES 使用量（例如 RawUsage、TRESMins、TRESRunMins）的浮點數。例如，如果使用因子是 2，作業每執行一個 TRESBillingUnit 秒將計為 2。如果使用因子是 .5，每秒只計為一半的時間。設定為 0 將不會從作業添加任何計時使用量。

使用因子僅適用於作業的 QOS，而不適用於分割區 QOS。

如果設定了 UsageFactorSafe 旗標且 AccountingStorageEnforce 包含 *Safe*，作業只有在應用 UsageFactor 後作業可以執行到完成時才能執行，且不會因限制被終止。

如果未設定 UsageFactorSafe 旗標且 AccountingStorageEnforce 包含 *Safe*，作業將能夠在未應用 UsageFactor 的情況下被排程，且不會因限制被終止。

如果未設定 UsageFactorSafe 旗標且 AccountingStorageEnforce 不包含 *Safe*，作業將在限制未達到時被排程，但可能因限制被終止。

詳見 slurm.conf man 頁面中的 [AccountingStorageEnforce](slurm.conf.html#OPT_AccountingStorageEnforce)。

**MaxNodes** 和 **MaxTime** 選項在 Slurm 的配置中已經按分割區存在，但上述選項提供了按使用者施加限制的能力。**MaxJobs** 選項為 Slurm 提供了一種全新的機制來控制任何個人可能對叢集施加的工作負載，以在使用者之間實現某種平衡。

當為用作分割區 QOS 的 QOS 分配限制時，請記住這些限制是在 QOS 級別強制執行的，而不是針對每個分割區單獨執行。例如，如果 QOS 定義了 **GrpTRES=cpu=20** 限制並且 QOS 被分配給兩個獨特的分割區，使用者將被限制為 QOS 的 20 個 CPU，而不是每個分割區 20 個 CPU。

公平份額排程基於 Slurm 資料庫中維護的階層帳號資料。更多資訊可以在 [priority/multifactor](priority_multifactor.md) 外掛程式描述中找到。

### GRES 的特定限制

當 GRES 有與之關聯的類型且限制應用於此特定類型（例如 *MaxTRESPerUser=gres/gpu:tesla=1*）時，如果使用者請求通用 gres，類型的限制將不會被強制執行。在這種情況下，額外的 lua 作業提交外掛程式來檢查使用者請求可能會變得有用。例如，如果請求 *--gres=gpu:2* 並設定了 *MaxTRESPerUser=gres/gpu:tesla=1* 的限制，限制不會被強制執行，因此仍然可以獲得兩個 tesla。

這是由於設計限制。強制執行這種限制的唯一方法是將限制的規格與強制使用者始終請求特定類型型號的作業提交外掛程式結合使用。

基本 lua 作業提交外掛程式函數的範例可能是：

```lua
function slurm_job_submit(job_desc, part_list, submit_uid)
   gres_request = ""
   t = {job_desc.tres_per_job,
        job_desc.tres_per_socket,
        job_desc.tres_per_task,
        job_desc.tres_per_node}
   for k in pairs(t) do
        gres_request = gres_request .. t[k] .. ","
   end
   if (gres_request ~= nil)
   then
      for g in gres_request:gmatch("[^,]+")
      do
         bad = string.match(g,'^gres/gpu[:=]*[0-9]*$')
         if (bad ~= nil)
         then
            slurm.log_info("User specified gpu GRES without type: %s", bad)
            slurm.user_msg("You must always specify a type when requesting gpu GRES")
            return slurm.ERROR
         end
      end
   end
end
```

有了這個腳本和限制，將強制使用者始終指定帶有類型的 gpu，從而強制執行每個特定型號的限制。

當為分割區定義 **TRESBillingWeights** 時，應包含有類型和無類型的資源。例如，如果一個分割區中有「tesla」GPU，而您只為「tesla」類型的 GPU 資源定義計費權重，則這些權重將不會應用於通用 GPU。

建議也為通用和特定 gres 類型設定 **AccountingStorageTRES**，否則請求通用 gres 實例的請求不會被會計。例如，要追蹤通用 GPU 和 Tesla GPU，您需要在 slurm.conf 中設定：

```
AccountingStorageTRES=gres/gpu,gres/gpu:tesla
```

詳見[可追蹤資源 TRES](tres.md)。

### 作業原因代碼

當排程器評估待處理作業但發現超過配置的資源限制時，將為作業分配相應的原因。更多詳細資訊可以在[作業原因代碼](job_reason_codes.md)頁面找到。有關排程的更多詳細資訊可以在[排程配置指南](sched_config.md)中找到。

---

## Explanation

### 限制類型比較

| 前綴 | 範圍 | 說明 |
|------|------|------|
| Grp* | 群組 | 關聯/QOS 及其子項的總計限制 |
| Max* | 個體 | 單一作業或使用者/帳號的限制 |

### 常用限制參數

| 參數 | 說明 | 應用於 |
|------|------|--------|
| GrpTRES | 群組可使用的總 TRES | 關聯、QOS |
| GrpJobs | 群組可同時執行的作業數 | 關聯、QOS |
| MaxTRESPerJob | 單一作業可使用的最大 TRES | 關聯、QOS |
| MaxTRESPerUser | 每使用者可使用的最大 TRES | QOS |
| MaxJobsPerUser | 每使用者可同時執行的作業數 | QOS |
| MaxWallDurationPerJob | 單一作業的最大執行時間 | 關聯、QOS |

---

## Practical Example

### 場景：配置階層式資源限制

研究中心需要限制各部門和使用者的資源使用。

```bash
# 1. 建立帳號階層
$ sacctmgr add account research Description="Research Department"
$ sacctmgr add account physics parent=research Description="Physics Lab"
$ sacctmgr add account chemistry parent=research Description="Chemistry Lab"

# 2. 設定部門級別限制（research 帳號）
$ sacctmgr modify account research set \
    GrpTRES=cpu=500,mem=2000G \
    GrpJobs=100 \
    MaxWallDurationPerJob=7-00:00:00
# GrpTRES=cpu=500,mem=2000G  研究部門總共最多 500 CPU、2TB 記憶體
# GrpJobs=100                 研究部門最多同時執行 100 個作業
# MaxWallDurationPerJob=...   單一作業最長執行 7 天

# 3. 設定實驗室級別限制
$ sacctmgr modify account physics set \
    GrpTRES=cpu=300,mem=1200G \
    GrpJobs=50
# 物理實驗室在研究部門限制內，最多 300 CPU

$ sacctmgr modify account chemistry set \
    GrpTRES=cpu=200,mem=800G \
    GrpJobs=50
# 化學實驗室最多 200 CPU

# 4. 添加使用者到帳號並設定使用者級別限制
$ sacctmgr add user alice account=physics
$ sacctmgr add user bob account=physics

$ sacctmgr modify user alice set \
    MaxTRESPerJob=cpu=50,mem=200G \
    MaxJobs=10
# MaxTRESPerJob=...  alice 單一作業最多 50 CPU、200G 記憶體
# MaxJobs=10         alice 最多同時執行 10 個作業

# 5. 查看關聯和限制
$ sacctmgr show assoc format=account,user,grptres,maxjobs,maxtresperjob
```

### 配置 QOS 限制

```bash
# 1. 建立 QOS 並設定限制
$ sacctmgr add qos standard
$ sacctmgr modify qos standard set \
    GrpTRES=cpu=100 \
    MaxTRESPerUser=cpu=20,mem=80G \
    MaxJobsPerUser=5 \
    MaxSubmitJobsPerUser=20
# GrpTRES=cpu=100        此 QOS 所有作業總共最多使用 100 CPU
# MaxTRESPerUser=...     每使用者最多 20 CPU、80G 記憶體
# MaxJobsPerUser=5       每使用者最多同時執行 5 個作業
# MaxSubmitJobsPerUser=20 每使用者最多提交 20 個作業（含待處理）

# 2. 分配 QOS 給使用者
$ sacctmgr modify user alice set qos=standard

# 3. 提交作業時的限制生效
$ sbatch -n 30 my_job.sh  # 如果超過 MaxTRESPerUser 將被拒絕或待處理
```

### 啟用限制強制執行

```bash
# 在 slurm.conf 中配置
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageEnforce=associations,limits,safe

# associations - 強制使用者必須有有效關聯
# limits       - 強制執行限制
# safe         - 確保作業能完成才啟動
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **忘記啟用限制強制執行**
   ```bash
   # 問題：設定了限制但不生效
   # 原因：沒有設定 AccountingStorageEnforce

   # 解決：在 slurm.conf 添加
   AccountingStorageEnforce=limits
   ```

2. **階層限制理解錯誤**
   ```bash
   # 問題：以為子帳號可以超過父帳號的限制
   # 父帳號：GrpTRES=cpu=100
   # 子帳號：GrpTRES=cpu=200  # 這不會生效！

   # 正確理解：子帳號受父帳號限制，200 會被限制在 100 以內
   ```

3. **GRES 類型限制不生效**
   ```bash
   # 問題：設定了 gres/gpu:tesla=1 限制但使用者請求 --gres=gpu:2 仍可獲得 2 個 tesla
   # 原因：通用 gres 請求不會觸發特定類型的限制

   # 解決：使用 job_submit 外掛程式強制指定類型
   ```

4. **分割區 QOS 與作業 QOS 混淆**
   ```bash
   # 問題：期望作業 QOS 覆蓋分割區 QOS 的限制
   # 預設行為：分割區 QOS 優先

   # 解決：在作業 QOS 設定 Flags=OverPartQOS
   $ sacctmgr modify qos myqos set Flags=OverPartQOS
   ```

### 實用建議

- **使用 safe 選項**：防止作業因限制在執行中被終止
  ```bash
  AccountingStorageEnforce=associations,limits,safe
  ```

- **監控限制使用情況**：
  ```bash
  # 查看使用者的使用量
  $ sshare -u alice

  # 查看待處理作業的原因
  $ squeue -u alice -t pending -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"
  ```

- **漸進式設定限制**：先寬鬆後收緊，避免突然中斷使用者工作

- **設定 DenyOnLimit 旗標**：讓使用者立即知道超出限制
  ```bash
  $ sacctmgr modify qos standard set Flags=DenyOnLimit
  ```

---

## Quick Reference

### AccountingStorageEnforce 選項

| 選項 | 說明 |
|------|------|
| associations | 要求有效的關聯 |
| limits | 強制執行限制 |
| qos | 要求有效的 QOS |
| safe | 確保作業能完成 |
| wckeys | 要求有效的 WCKey |

### 限制設定指令

| 指令 | 說明 |
|------|------|
| `sacctmgr modify user <name> set ...` | 設定使用者限制 |
| `sacctmgr modify account <name> set ...` | 設定帳號限制 |
| `sacctmgr modify qos <name> set ...` | 設定 QOS 限制 |
| `sacctmgr show assoc` | 查看關聯和限制 |

### 常用限制參數速查

| 參數 | 層級 | 說明 |
|------|------|------|
| `GrpTRES=` | 帳號/QOS | 群組 TRES 總量 |
| `GrpJobs=` | 帳號/QOS | 群組作業數 |
| `MaxTRESPerJob=` | 關聯/QOS | 單作業最大 TRES |
| `MaxTRESPerUser=` | QOS | 每使用者最大 TRES |
| `MaxJobsPerUser=` | QOS | 每使用者作業數 |
| `MaxWallDurationPerJob=` | 關聯/QOS | 最大執行時間 |

### 作業原因代碼（限制相關）

| 原因 | 說明 |
|------|------|
| AssocGrpCPUMinutes | 關聯群組 CPU 分鐘限制 |
| AssocGrpCPURunMinutes | 關聯群組執行中 CPU 分鐘限制 |
| AssocMaxJobsLimit | 關聯最大作業數限制 |
| QOSGrpCPULimit | QOS 群組 CPU 限制 |
| QOSMaxCpuPerUserLimit | QOS 每使用者最大 CPU 限制 |
