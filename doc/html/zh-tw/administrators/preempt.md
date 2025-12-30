# Slurm 作業搶占

## TL;DR
- **搶占 (Preemption)** 可「停止」低優先級作業讓高優先級作業執行
- 搶占模式：`CANCEL`（取消）、`REQUEUE`（重新排隊）、`SUSPEND`（暫停）
- 使用 `PreemptType` 設定搶占類型：`partition_prio`（分割區優先級）或 `qos`（服務品質）
- 分割區的 `PriorityTier` 越高，該分割區的作業越能搶占其他作業
- `SUSPEND` 模式需要配合 `GANG` 排程才能恢復被暫停的作業

---

## Translation

### 概述

Slurm 支援作業搶占 (Job Preemption)，即「停止」一個或多個「低優先級」作業以讓「高優先級」作業執行的行為。作業搶占是作為 Slurm [幫派排程](gang_scheduling.html) (Gang Scheduling) 邏輯的變體實作的。當一個可以搶占其他作業的作業被分配到已經分配給一個或多個可被第一個作業搶占的作業的資源時，可搶占的作業會被搶占。根據配置，被搶占的作業可以被取消，或者可以重新排隊並使用其他資源啟動，或者暫停並在搶占者作業完成後恢復，甚至可以使用幫派排程與搶占者共享資源。

作業的分割區 (Partition) 的 PriorityTier 或其服務品質 (Quality Of Service, QOS) 可用於識別哪些作業可以搶占或被其他作業搶占。Slurm 提供按分割區或按 QOS 配置搶占機制的能力。例如，低優先級佇列中的作業可能會被重新排隊，而中優先級佇列中的作業可能會被暫停。

### 配置

有幾個與搶占相關的重要配置參數：

**SelectType**：Slurm 作業搶占邏輯支援由 *select/linear* 外掛程式分配的節點，以及由 *select/cons_tres* 外掛程式分配的插槽/核心/CPU 資源。

**SelectTypeParameter**：由於資源可能會被作業過度分配（暫停的作業仍留在記憶體中），資源選擇外掛程式應配置為追蹤每個作業使用的記憶體量，以確保不會發生記憶體分頁交換。當選擇 *select/linear* 時，我們建議設定 *SelectTypeParameter=CR_Memory*。當選擇 *select/cons_tres* 時，我們建議將記憶體作為資源（例如 *SelectTypeParameter=CR_Core_Memory*）。
**注意**：除非 *PreemptMode=SUSPEND,GANG*，否則這些記憶體管理參數不是關鍵的。

**DefMemPerCPU**：由於作業請求可能沒有明確指定記憶體需求，我們還建議配置 *DefMemPerCPU*（每個分配的 CPU 的預設記憶體）或 *DefMemPerNode*（每個分配的節點的預設記憶體）。在 *slurm.conf* 中配置 *MaxMemPerCPU*（每個分配的 CPU 的最大記憶體）或 *MaxMemPerNode*（每個分配的節點的最大記憶體）也可能是需要的。使用者可以在作業提交時使用 *--mem* 或 *--mem-per-cpu* 選項來指定其記憶體需求。
**注意**：除非 *PreemptMode=SUSPEND,GANG*，否則這些記憶體管理參數不是關鍵的。

**GraceTime**：指定作業在被選中搶占後執行的時間段。此選項可以分別使用 *slurm.conf* 檔案或資料庫按分割區或 QOS 指定。*GraceTime* 以秒為單位指定，預設值為零，這表示沒有搶占延遲。一旦作業被選中進行搶占，其結束時間將設定為當前時間加上 *GraceTime*。作業會立即收到 SIGCONT 和 SIGTERM 訊號，以通知其即將終止。隨後在達到新的結束時間時發送 SIGCONT、SIGTERM 和 SIGKILL 訊號序列。
**注意**：當配置 *PreemptMode=SUSPEND* 或使用 *scontrol suspend* 暫停作業時，不使用此參數。在這些情況下設定搶占寬限時間，請參閱 [suspend_grace_time](slurm.conf.html#OPT_suspend_grace_time)。

**JobAcctGatherType 和 JobAcctGatherFrequency**：系統會為每個作業配置「最大資料段大小」和「最大虛擬記憶體大小」系統限制，以確保作業不會超過其請求的記憶體量。如果您希望啟用額外的記憶體限制強制執行，請使用 *JobAcctGatherType* 和 *JobAcctGatherFrequency* 參數配置作業會計。當啟用會計且作業超過其配置的記憶體限制時，它將被取消以防止對共享相同資源的其他作業產生不利影響。
**注意**：除非 *PreemptMode=SUSPEND,GANG*，否則這些記憶體管理參數不是關鍵的。

**PreemptMode**：用於搶占作業或啟用幫派排程的機制。當 *PreemptType* 參數設定為啟用搶占時，slurm.conf 主區段中的 *PreemptMode* 選擇用於叢集的搶占可搶占作業的預設機制。

如果 *PreemptType=preempt/partition_prio*，可以按分割區指定 *PreemptMode* 以覆蓋此預設值。或者，如果 *PreemptType=preempt/qos*，可以按 QOS 指定它。在任何一種情況下，當啟用搶占時，必須為整個叢集指定有效的預設 *PreemptMode* 值。

*GANG* 選項用於獨立於是否啟用搶占（即獨立於 *PreemptType*）來啟用幫派排程。它可以與其他 *PreemptMode* 設定一起指定，兩個選項用逗號分隔（例如 *PreemptMode=SUSPEND,GANG*）。

- **OFF**：是預設值，禁用作業搶占和幫派排程。它只與 *PreemptType=preempt/none* 相容。

- **CANCEL**：被搶占的作業將被取消。

- **GANG**：啟用同一分割區中作業的幫派排程（時間分片），並允許恢復暫停的作業。要使用幫派排程，必須在叢集級別指定 **GANG** 選項。
  **注意**：如果 **GANG** 排程與 *PreemptType=preempt/partition_prio* 一起啟用，控制器將忽略 *PreemptExemptTime* 和以下 *PreemptParameters*：*reorder_count*、*strict_order* 和 *youngest_first*。
  **注意**：幫派排程是針對每個分割區獨立執行的，因此如果您只想透過 *OverSubscribe* 進行時間分片而不進行任何搶占，則不建議配置具有重疊節點的分割區。另一方面，如果您想使用 *PreemptType=preempt/partition_prio* 允許來自較高 PriorityTier 分割區的作業暫停來自較低 PriorityTier 分割區的作業，則您需要重疊的分割區，並使用 *PreemptMode=SUSPEND,GANG* 來使用幫派排程器恢復暫停的作業。在任何一種情況下，不同分割區上的作業之間不會發生時間分片。

- **REQUEUE**：透過重新排隊（如果可能）或取消作業來搶占作業。要重新排隊作業，它們必須設定「--requeue」sbatch 選項，或者 slurm.conf 中的叢集範圍 JobRequeue 參數必須設定為 1。

- **SUSPEND**：被搶占的作業將被暫停，稍後幫派排程器將恢復它們。因此，*SUSPEND* 搶占模式始終需要在叢集級別指定 *GANG* 選項。此外，因為暫停的作業仍將在分配的節點上使用記憶體，Slurm 需要能夠追蹤記憶體資源才能暫停作業。
  **注意**：因為幫派排程是針對每個分割區獨立執行的，如果使用 *PreemptType=preempt/partition_prio*，則較高 PriorityTier 分割區中的作業將暫停較低 PriorityTier 分割區中的作業以在釋放的資源上執行。只有當搶占者作業結束時，暫停的作業才會被幫派排程器恢復。如果配置了 *PreemptType=preempt/qos* 並且被搶占的作業和搶占者作業在同一分割區上，則它們將與幫派排程器共享資源（時間分片）。如果不是（即被搶占者和搶占者在不同的分割區上），則被搶占的作業將保持暫停狀態直到搶占者結束。

**PreemptType**：指定用於識別哪些作業可以被搶占以啟動待處理作業的外掛程式。

- **preempt/none**：作業搶占被禁用（預設）。
- **preempt/partition_prio**：作業搶占基於分割區 *PriorityTier*。較高 PriorityTier 分割區中的作業可以搶占較低 PriorityTier 分割區中的作業。這與 *PreemptMode=OFF* 不相容。
- **preempt/qos**：作業搶占規則由 Slurm 資料庫中的服務品質 (QOS) 規格指定。在 *PreemptMode=SUSPEND* 的情況下，搶占作業需要提交到具有較高 PriorityTier 的分割區或同一分割區。此選項與 *PreemptMode=OFF* 不相容。*PreemptMode=SUSPEND* 的配置僅由 *SelectType=select/cons_tres* 外掛程式支援。請參閱 [sacctmgr man 頁面](sacctmgr.html) 以配置 *preempt/qos* 的選項。

**PreemptExemptTime**：指定作業被考慮搶占前的最短執行時間。這僅在 *PreemptMode* 設定為 *REQUEUE* 或 *CANCEL* 時有效。它以時間字串指定：-1 的時間禁用此選項，相當於 0。可接受的時間格式包括「minutes」、「minutes:seconds」、「hours:minutes:seconds」、「days-hours」、「days-hours:minutes」和「days-hours:minutes:seconds」。PreemptEligibleTime 顯示在「scontrol show job <作業 id>」的輸出中。

**PriorityTier**：配置分割區的 *PriorityTier* 設定相對於其他分割區，以在 *PreemptType=preempt/partition_prio* 時控制搶占行為。如果來自兩個不同分割區的兩個作業被分配到相同的資源，具有較大 *PriorityTier* 值的分割區中的作業將搶占具有較小 *PriorityTier* 值的分割區中的作業。如果兩個分割區的 *PriorityTier* 值相等，則不會發生搶占。預設 *PriorityTier* 值為 1。
**注意**：除了用於基於分割區的搶占外，*PriorityTier* 也對排程有影響。排程器將在評估其他分割區中的作業之前，先評估具有最高 *PriorityTier* 的分割區中的作業，無論哪些作業具有最高優先級。排程器在評估具有相同 *PriorityTier* 的分割區中的作業時將考慮作業優先級。

**OverSubscribe**：對於使用暫停/恢復機制進行作業搶占的所有分割區，將分割區的 *OverSubscribe* 設定配置為 *FORCE*。*FORCE* 選項支援一個額外的參數，用於控制多少作業可以過度訂閱計算資源（FORCE[:max_share]）。預設 max_share 值為 4。為了搶占作業（而不是幫派排程它們），始終將 max_share 設定為 1。要允許此分割區中最多 2 個作業被分配到公共資源（並進行幫派排程），請設定 *OverSubscribe=FORCE:2*。
**注意**：如果透過作業搶占啟動，*PreemptType=preempt/qos* 將允許在分割區上執行一個額外的作業。例如，*OverSubscribe=FORCE:1* 的配置通常只允許每個資源一個作業，但如果透過基於 QOS 的搶占完成，可以啟動第二個作業。

**ExclusiveUser**：在 *ExclusiveUser=YES* 的分割區中，作業將被阻止搶占或被任何其他使用者的任何作業搶占。唯一的例外是這些 ExclusiveUser 作業將能夠搶占（但不能被搶占）其他使用者的完全「--exclusive」作業。這與「--exclusive=user」阻止搶占的原因相同，但此分割區級別設定只能透過使作業完全獨佔來覆蓋。

**MCSParameters**：如果作業上設定了 [MCS](mcs.html) 標籤，搶占將限制為具有相同 MCS 標籤的其他作業。如果此參數配置為使用 `enforced,select`，MCS 標籤將預設設定在作業上，導致此限制是普遍的。

要在進行上述配置更改後啟用搶占，如果 Slurm 已在執行，請重新啟動它。Slurm 中外掛程式設定的任何更改都需要完全重新啟動守護程序。如果您只更改分割區 *PriorityTier* 或 *OverSubscribe* 設定，可以使用 *scontrol reconfig* 更新。

如果作業請求透過使用「--exclusive=user」或「--exclusive=mcs」作業選項限制 Slurm 在節點上執行多個使用者或帳號的作業的能力，作業將被阻止搶占或被任何與使用者或 MCS 不匹配的作業搶占。唯一的例外是這些 exclusive=user 作業將能夠搶占（但不能被搶占）其他使用者的完全「--exclusive」作業。如果使用搶占，通常建議透過使用 job_submit 外掛程式禁用「--exclusive=user」和「--exclusive=mcs」作業選項（將「job_desc.shared」的值設定為「NO_VAL16」）。

對於異質作業要被考慮搶占，所有組件都必須有資格被搶占。當異質作業要被搶占時，具有最高順序 PreemptMode 的作業的第一個識別的組件（*SUSPEND*（最高）、*REQUEUE*、*CANCEL*（最低））將用於設定所有組件的 PreemptMode。異質作業每個組件的 *GraceTime* 和使用者警告訊號保持唯一。

因為當作業暫停時授權不會被釋放，使用高優先級作業請求的授權的作業只有在 PreemptMode 為 *REQUEUE* 或 *CANCEL* 且設定了 *PreemptParameters=reclaim_licenses* 時才會被搶占。

### 搶占設計和操作

*SelectType* 外掛程式將識別待處理作業可以開始執行的資源。當 *PreemptMode* 配置為 *CANCEL*、*SUSPEND* 或 *REQUEUE* 時，選擇外掛程式還將根據需要搶占執行中的作業以啟動待處理作業。當 *PreemptMode=SUSPEND,GANG* 時，選擇外掛程式將啟動待處理作業並依賴幫派排程邏輯執行作業暫停和恢復，如下所述。

選擇外掛程式會為每個待開始的待處理作業傳遞一個有序的可搶占作業列表。此列表按以下方式排序：

1. QOS 優先級，
2. 分割區優先級和作業大小（以優先搶占較小的作業），或
3. 作業開始時間（使用 *PreemptParameters=youngest_first*）。

選擇外掛程式將確定待處理作業是否可以在不搶占任何作業的情況下啟動，如果是，則使用可用資源啟動作業。否則，選擇外掛程式將模擬按優先級順序列表中每個作業的搶占，並在每次搶占後測試作業是否可以啟動。一旦作業可以啟動，搶占佇列中的較高優先級作業將不被考慮，但原始列表中要搶占的作業可能不是最佳的。例如，要啟動一個 8 節點作業，有序的搶占候選者可能是 2 節點、4 節點和 8 節點。搶占所有三個作業將允許待處理作業啟動，但透過重新排序搶占候選者，可以在僅搶占一個作業後啟動待處理作業。為了解決這個問題，搶占候選者被重新排序，需要搶占的最後一個作業放在列表的第一位，所有其他要搶占的作業按其分配中與為待處理作業選擇的資源重疊的節點數排序。在上面的例子中，8 節點作業將被移到列表的第一位。然後將重複模擬按優先級順序列表中每個作業的搶占過程，以做出搶占哪些作業的最終決定。這個兩階段過程可能會搶占不完全按搶占優先級順序的作業，但搶占的作業將比原本需要的少。請參閱 PreemptParameters 配置參數的 *reorder_count* 和 *strict_order* 選項以獲取搶占調整參數。

當啟用時，幫派排程邏輯（也支援作業搶占）會追蹤分配給所有作業的資源。對於每個分割區，維護一個「活動點陣圖」來追蹤 Slurm 叢集中所有同時執行的作業。每個分割區還維護該分割區的作業列表和「影子」作業列表。「影子」作業是高優先級作業分配，它們在低優先級作業的活動點陣圖上「投下陰影」。被這些「陰影」捕獲的作業將被搶占。

每次將新作業分配到分割區中的資源並開始執行時，幫派排程器會將此作業的「影子」添加到所有較低優先級的分割區。然後重建這些較低優先級分割區的活動點陣圖，首先添加影子作業。任何被一個或多個「影子」作業替換的現有作業都會被暫停（搶占）。相反，當高優先級執行中的作業完成時，其「影子」消失，較低優先級分割區的活動點陣圖被重建，以查看是否可以恢復任何暫停的作業。

幫派排程器外掛程式被設計為對「select」外掛程式做出的資源分配決策*被動反應*。「select」外掛程式已被增強，以在配置作業搶占時識別，並在為作業選擇資源時考慮每個分割區的優先級。為每個作業選擇資源時，選擇器會避免被其他作業使用的資源（除非已配置共享，在這種情況下會進行一些負載平衡）。然而，當啟用作業搶占時，即使在這些分割區中禁用共享，選擇外掛程式也可能選擇已被具有較低優先級設定的分割區中的作業使用的資源。

這使幫派排程器負責控制哪些作業應該在過度分配的資源上執行。如果 *PreemptMode=SUSPEND*，作業使用支援 *scontrol suspend* 和 *scontrol resume* 的相同內部函數暫停。觀察幫派排程器操作的好方法是在終端機視窗中執行 *squeue -i<時間>*。

### 回填排程期間搶占的限制

出於效能原因，回填排程器為作業保留整個節點，而不是部分節點。如果在回填排程期間作業搶占一個或多個其他作業，那些被搶占作業的整個節點都會為搶占者作業保留，即使搶占者作業請求的資源少於此數量。這些保留的節點在該回填週期中對其他作業不可用，即使其他作業可以放在這些節點上。因此，作業在單次回填迭代中可能搶占比其請求更多的資源。

---

## Explanation

### 搶占模式比較

| 模式 | 動作 | 作業狀態 | 需要 GANG | 記憶體追蹤 |
|------|------|----------|-----------|------------|
| OFF | 無搶占 | - | 否 | 否 |
| CANCEL | 直接取消作業 | 終止 | 否 | 否 |
| REQUEUE | 重新排隊作業 | 待處理 | 否 | 否 |
| SUSPEND | 暫停作業 | 暫停 | 是 | 是 |
| GANG | 時間分片 | 輪流執行 | 是 | 是 |

### 搶占類型

| PreemptType | 搶占依據 | 說明 |
|-------------|----------|------|
| preempt/none | 無 | 禁用搶占（預設）|
| preempt/partition_prio | 分割區 PriorityTier | 高優先級分割區搶占低優先級分割區 |
| preempt/qos | QOS 設定 | 根據 QOS 搶占規則決定 |

---

## Practical Example

### 場景：配置三層優先級分割區搶占

研究中心想要設定三個分割區：低優先級（學生使用）、中優先級（研究人員）、高優先級（緊急計算）。

```bash
# slurm.conf 配置
PreemptType=preempt/partition_prio
PreemptMode=REQUEUE              # 預設搶占模式為重新排隊

# 分割區配置
# 低優先級分割區 - 學生作業，被搶占時重新排隊
PartitionName=student Nodes=node[01-20] Default=YES \
    PriorityTier=10 PreemptMode=requeue OverSubscribe=NO

# 中優先級分割區 - 研究作業，被搶占時暫停
PartitionName=research Nodes=node[01-20] Default=NO \
    PriorityTier=20 PreemptMode=suspend OverSubscribe=FORCE:1

# 高優先級分割區 - 緊急作業，不會被搶占
PartitionName=urgent Nodes=node[01-20] Default=NO \
    PriorityTier=30 PreemptMode=off OverSubscribe=FORCE:1
```

### 操作示範

```bash
# 1. 學生提交作業到低優先級分割區
$ sbatch -p student -N4 --requeue student_job.sh
# --requeue     允許作業被重新排隊（搶占時需要）
Submitted batch job 100

# 2. 研究人員提交作業到中優先級分割區
$ sbatch -p research -N4 research_job.sh
Submitted batch job 101

# 3. 查看作業狀態
$ squeue
JOBID PARTITION     NAME     USER  ST   TIME  NODES NODELIST
  100   student   stu_job  student  PD   0:00      4 (Resources)
  101  research   res_job   prof_a   R   0:30      4 node[01-04]
# 作業 100 因資源不足待處理，作業 101 執行中

# 4. 管理員提交緊急作業
$ sbatch -p urgent -N4 urgent_calc.sh
Submitted batch job 102

# 5. 再次查看狀態 - 研究作業被暫停
$ squeue
JOBID PARTITION     NAME     USER  ST   TIME  NODES NODELIST
  102    urgent urgent_cal   admin   R   0:05      4 node[01-04]
  101  research   res_job   prof_a   S   0:30      4 node[01-04]
  100   student   stu_job  student  PD   0:00      4 (Resources)
# 作業 101 狀態變為 S (Suspended)，被緊急作業搶占
# 作業 102 正在執行

# 6. 緊急作業完成後，研究作業恢復
$ squeue
JOBID PARTITION     NAME     USER  ST   TIME  NODES NODELIST
  101  research   res_job   prof_a   R   0:45      4 node[01-04]
  100   student   stu_job  student  PD   0:00      4 (Resources)
```

### 使用 QOS 搶占

```bash
# slurm.conf 配置
PreemptType=preempt/qos
PreemptMode=CANCEL

# 使用 sacctmgr 配置 QOS 搶占關係
# 建立 QOS
$ sacctmgr add qos low
$ sacctmgr add qos normal
$ sacctmgr add qos high

# 設定 high QOS 可搶占 low 和 normal
$ sacctmgr modify qos high set preempt=low,normal

# 設定 normal QOS 可搶占 low
$ sacctmgr modify qos normal set preempt=low

# 提交帶有 QOS 的作業
$ sbatch --qos=high my_job.sh
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **SUSPEND 模式沒有配合 GANG**
   ```bash
   # 錯誤：只設 SUSPEND 沒有 GANG
   PreemptMode=SUSPEND
   # 被暫停的作業無法恢復！

   # 正確：SUSPEND 必須配合 GANG
   PreemptMode=SUSPEND,GANG
   ```

2. **忘記啟用 --requeue 選項**
   ```bash
   # 錯誤：作業沒有設定 --requeue
   $ sbatch my_job.sh
   # PreemptMode=REQUEUE 時作業會被取消而非重新排隊

   # 正確：啟用重新排隊
   $ sbatch --requeue my_job.sh
   # 或在 slurm.conf 設定 JobRequeue=1
   ```

3. **SUSPEND 模式沒有追蹤記憶體**
   ```bash
   # 問題：暫停的作業仍佔用記憶體，可能導致記憶體不足

   # 解決：配置記憶體追蹤
   SelectTypeParameter=CR_Core_Memory
   DefMemPerCPU=2048
   ```

4. **overlapping 分割區配置不當**
   ```bash
   # 問題：想用時間分片但不同分割區間不會發生
   # 幫派排程只在同一分割區內的作業間進行

   # 解決：確保需要時間分片的作業在同一分割區
   ```

### 實用建議

- **設定 GraceTime**：給被搶占作業一些時間清理和儲存狀態
  ```bash
  # 分割區級別
  PartitionName=batch GraceTime=60

  # QOS 級別（透過資料庫）
  $ sacctmgr modify qos normal set gracetime=120
  ```

- **使用 PreemptExemptTime**：避免剛啟動的作業立即被搶占
  ```bash
  PreemptExemptTime=5:00  # 作業執行 5 分鐘後才能被搶占
  ```

- **監控搶占情況**：
  ```bash
  # 查看作業是否有搶占資格
  $ scontrol show job <jobid> | grep PreemptEligibleTime

  # 查看被搶占的作業
  $ sacct -j <jobid> --format=JobID,State,ExitCode
  ```

- **異質作業注意事項**：所有組件都必須可被搶占，否則整個作業不會被搶占

---

## Quick Reference

### 配置參數

| 參數 | 說明 |
|------|------|
| `PreemptType` | 搶占類型（none/partition_prio/qos）|
| `PreemptMode` | 搶占模式（OFF/CANCEL/REQUEUE/SUSPEND/GANG）|
| `PreemptExemptTime` | 搶占豁免時間 |
| `PriorityTier` | 分割區優先級層級 |
| `GraceTime` | 搶占寬限時間 |
| `OverSubscribe` | 過度訂閱設定 |

### 作業狀態

| 狀態碼 | 說明 |
|--------|------|
| R | Running - 執行中 |
| S | Suspended - 被暫停 |
| PD | Pending - 待處理/被重新排隊 |

### 常用指令

| 指令 | 說明 |
|------|------|
| `scontrol suspend <jobid>` | 手動暫停作業 |
| `scontrol resume <jobid>` | 手動恢復作業 |
| `scontrol show job <jobid>` | 查看作業詳情（含搶占資訊）|
| `sacctmgr show qos` | 查看 QOS 設定 |
| `sacctmgr modify qos <name> set preempt=<qos_list>` | 設定 QOS 搶占關係 |
