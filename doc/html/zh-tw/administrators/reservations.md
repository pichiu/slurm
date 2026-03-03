# Slurm 進階資源預約指南

## TL;DR
- **預約 (Reservation)** 可為特定使用者/帳號/QOS/分割區保留資源（核心、節點、授權、突發緩衝區）
- 使用 `scontrol create reservation` 建立預約，`scontrol show res` 查看
- 作業必須用 `--reservation=<名稱>` 指定預約才能使用預留的資源
- 支援維護預約、每日重複、浮動時間、彈性預約等多種模式
- 預約結束後會自動清除，也可手動刪除

---

## Translation

### 概述

Slurm 具備為特定使用者和/或 QOS 和/或分割區和/或特定帳號執行的作業保留資源的能力。資源預約 (Resource Reservation) 會識別該預約中的資源以及預約可用的時間段。可以保留的資源包括核心 (cores)、節點 (nodes)、授權 (licenses) 和/或突發緩衝區 (burst buffers)。包含節點或核心的預約會關聯到一個分割區，且無法跨越多個分割區的資源。唯一的例外是當預約是以明確請求的節點建立時。請注意，資源預約與 Slurm 的幫派排程器 (gang scheduler) 外掛程式不相容，因為無法準確預測執行中作業的終止時間。

請注意，預約的突發緩衝區和授權的處理方式與預約的核心或節點略有不同。當核心或節點被預約時，使用該預約的作業只能使用這些資源（此行為可透過 FLEX 旗標更改），且其他作業無法使用這些資源。預約的突發緩衝區和授權只能由與該預約關聯的作業使用，但未明確預約的授權可供任何作業使用。這消除了在建立的每個進階預約中明確放入授權的需要。

預約只能由 root 使用者或配置的 *SlurmUser* 使用 *scontrol* 指令來建立、更新或刪除。*scontrol* 和 *sview* 指令可用於查看預約。此外，root 和配置的 *SlurmUser* 可以存取所有預約，即使他們通常沒有存取權限。各個指令的 man 頁面包含詳細資訊。

### 預約建立

預約的一種常見操作模式是在特定時間為系統停機保留整個電腦。以下範例顯示在 2 月 6 日 16:00 建立全系統預約，持續 120 分鐘。「maint」旗標用於在會計目的上識別該預約為系統維護。「ignore_jobs」旗標用於表示在建立此預約時可以忽略目前執行中的作業。預設情況下，只有在開始時間不預期有執行中作業的資源才能被預約（所有執行中作業的時間限制將已達到）。在這種情況下，我們可以根據需要手動取消執行中的作業以執行系統維護。隨著預約時間接近，只有能在預約時間前完成的作業才會被啟動。

```bash
$ scontrol create reservation starttime=2009-02-06T16:00:00 \
   duration=120 user=root flags=maint,ignore_jobs nodes=ALL
Reservation created: root_3

$ scontrol show reservation
ReservationName=root_3 StartTime=2009-02-06T16:00:00
   EndTime=2009-02-06T18:00:00 Duration=120
   Nodes=ALL NodeCnt=20
   Features=(null) PartitionName=(null)
   Flags=MAINT,SPEC_NODES,IGNORE_JOBS Licenses=(null)
   BurstBuffers=(null)
   Users=root Accounts=(null)
```

這種做法的一個變體是配置授權來代表系統資源，例如全域檔案系統。系統資源可能實際上不需要授權才能使用，但 Slurm 授權可用於防止需要該資源的作業在該資源不可用時被啟動。可以為所有這些授權建立預約以對該資源執行維護。在以下範例中，我們為名為「lustre」的 1000 個授權建立預約。如果此叢集中配置了總共 1000 個 lustre 授權，此預約將防止任何指定需要 lustre 授權的作業在此預約期間被排程到此叢集上。

```bash
$ scontrol create reservation starttime=2009-04-06T16:00:00 \
   duration=120 user=root flags=license_only \
   licenses=lustre:1000
Reservation created: root_4

$ scontrol show reservation
ReservationName=root_4 StartTime=2009-04-06T16:00:00
   EndTime=2009-04-06T18:00:00 Duration=120
   Nodes= NodeCnt=0
   Features=(null) PartitionName=(null)
   Flags=LICENSE_ONLY Licenses=lustre*1000
   BurstBuffers=(null)
   Users=root Accounts=(null)
```

另一種操作模式是為無限期保留特定節點以研究這些節點上的問題。這也可以使用專門用於此目的的 Slurm 分割區來完成，但這將無法捕獲其維護性質的使用。

```bash
$ scontrol create reservation user=root starttime=now \
   duration=infinite flags=maint nodes=sun000
Reservation created: root_5

$ scontrol show res
ReservationName=root_5 StartTime=2009-02-04T16:22:57
   EndTime=2009-02-04T16:21:57 Duration=4294967295
   Nodes=sun000 NodeCnt=1
   Features=(null) PartitionName=(null)
   Flags=MAINT,SPEC_NODES Licenses=(null)
   BurstBuffers=(null)
   Users=root Accounts=(null)
```

下一個範例是在預設 Slurm 分割區中保留十個節點，從中午開始，持續 60 分鐘，每天重複。預約只對使用者「alan」和「brenda」可用。

```bash
$ scontrol create reservation user=alan,brenda \
   starttime=noon duration=60 flags=daily nodecnt=10
Reservation created: alan_6

$ scontrol show res
ReservationName=alan_6 StartTime=2009-02-05T12:00:00
   EndTime=2009-02-05T13:00:00 Duration=60
   Nodes=sun[000-003,007,010-013,017] NodeCnt=10
   Features=(null) PartitionName=pdebug
   Flags=DAILY Licenses=(null) BurstBuffers=(null)
   Users=alan,brenda Accounts=(null)
```

下一個範例是保留 100GB 的突發緩衝區空間，從今天中午開始，持續 60 分鐘。預約只對使用者「alan」和「brenda」可用。

```bash
$ scontrol create reservation user=alan,brenda \
   starttime=noon duration=60 flags=any_nodes burstbuffer=100GB
Reservation created: alan_7

$ scontrol show res
ReservationName=alan_7 StartTime=2009-02-05T12:00:00
   EndTime=2009-02-05T13:00:00 Duration=60
   Nodes= NodeCnt=0
   Features=(null) PartitionName=(null)
   Flags=ANY_NODES Licenses=(null) BurstBuffer=100GB
   Users=alan,brenda Accounts=(null)
```

請注意，與預約關聯的特定節點會在建立預約後立即識別。這允許使用者將檔案暫存到節點上，為預約期間的使用做準備。請注意，預約建立請求也可以識別從中選擇節點的分割區或每個選定節點必須包含的*一個*特性。

在較小的系統上，可能想要保留核心而不是整個節點。此功能允許管理員識別每個節點上要保留的核心數量，如以下範例所示。
**注意**：當系統配置為使用 select/linear 外掛程式時，核心預約不可用。

```bash
# 為使用者 alan 建立兩個核心的預約
$ scontrol create reservation StartTime=now Duration=60 \
  NodeCnt=1 CoreCnt=2 User=alan

# 為使用者 brenda 建立預約，在節點 tux8 上有兩個核心，
# 在節點 tux9 上有 4 個核心
$ scontrol create reservation StartTime=now Duration=60 \
  Nodes=tux8,tux9 CoreCnt=2,4 User=brenda
```

預約不僅可以為特定帳號和/或 QOS 和/或分割區和/或使用者的使用而建立，還可以防止特定帳號和/或 QOS 和/或分割區和/或使用者使用它們。在以下範例中，為帳號「foo」建立預約，但防止使用者「alan」使用該預約，即使使用帳號「foo」也是如此。

```bash
$ scontrol create reservation account=foo \
   user=-alan partition=pdebug \
   starttime=noon duration=60 nodecnt=2k,2k
Reservation created: alan_9

$ scontrol show res
ReservationName=alan_9 StartTime=2011-12-05T13:00:00
   EndTime=2011-12-05T14:00:00 Duration=60
   Nodes=bgp[000x011,210x311] NodeCnt=4096
   Features=(null) PartitionName=pdebug
   Flags= Licenses=(null) BurstBuffers=(null)
   Users=-alan Accounts=foo
```

建立預約時，您可以透過指定 **PartitionName** 選項請求 Slurm 包含分割區中的所有節點。如果您只想要該分割區中一定數量的節點或 CPU，可以將 **PartitionName** 與 **CoreCnt**、**NodeCnt** 或 **TRES** 選項結合使用以指定您想要多少資源。在以下範例中，在 'gpu' 分割區中建立預約，使用 **TRES** 選項將預約限制為 24 個處理器，分佈在 4 個節點上。

```bash
$ scontrol create reservationname=test start=now duration=1 \
   user=user1 partition=gpu tres=cpu=24,node=4
Reservation created: test

$ scontrol show res
ReservationName=test StartTime=2020-08-28T11:07:09
   EndTime=2020-08-28T11:08:09 Duration=00:01:00
   Nodes=node[01-04] NodeCnt=4 CoreCnt=24
   Features=(null) PartitionName=gpu
     NodeName=node01 CoreIDs=0-5
     NodeName=node02 CoreIDs=0-5
     NodeName=node03 CoreIDs=0-5
     NodeName=node04 CoreIDs=0-5
   TRES=cpu=24
   Users=user1 Accounts=(null) Licenses=(null)
   State=ACTIVE BurstBuffer=(null)
   MaxStartDelay=(null)
```

在 25.05 版本之前，如果預約使用基於使用者或群組的存取控制，且 slurmctld 無法驗證使用者或群組，它會刪除預約。從 25.05 版本開始，此行為已更改為保留預約並等待使用者/群組能夠再次被驗證，使預約能夠在臨時驗證中斷期間保持彈性。

### 預約使用

預約建立回應包含預約的名稱。此名稱由 Slurm 根據第一個使用者或帳號名稱和數字後綴自動產生。為了使用預約，作業提交請求必須明確指定該預約名稱。作業必須完全包含在命名的預約中。作業將在預約達到其 EndTime 後被取消。如果讓作業在預約 EndTime 後繼續執行，可以在 slurm.conf 中設定配置選項 *ResvOverRun* 來控制作業可以繼續執行多長時間。

```bash
$ sbatch --reservation=alan_6 -N4 my.script
sbatch: Submitted batch job 65540
```

請注意，使用預約不會改變作業的優先級，但它確實作為作業優先級的增強。任何具有預約的作業在排程到資源時，都會在同一 Slurm 分割區（佇列）中未與預約關聯的任何其他作業之前被考慮。

### 預約修改

預約可以由 root 使用者根據需要修改。例如，可以更改其持續時間或授予存取權限的使用者，如下所示：

```bash
$ scontrol update ReservationName=root_3 \
   duration=150 users=admin
Reservation updated.

$ scontrol show ReservationName=root_3
ReservationName=root_3 StartTime=2009-02-06T16:00:00
   EndTime=2009-02-06T18:30:00 Duration=150
   Nodes=ALL NodeCnt=20 Features=(null)
   PartitionName=(null) Flags=MAINT,SPEC_NODES
   Licenses=(null) BurstBuffers=(null)
   Users=admin Accounts=(null)
```

### 預約刪除

預約在其結束時間後會自動清除。它們也可以如下所示手動刪除。請注意，當有作業在預約中執行時，無法刪除該預約。

```bash
$ scontrol delete ReservationName=alan_6
```

**注意**：預設情況下，當預約結束時，預約請求將從提交到預約的任何待處理作業中移除，並將其置於保留狀態。使用 NO_HOLD_JOBS_AFTER_END 預約旗標讓作業在預約結束後在預約之外執行。

### 重疊預約

預設情況下，預約不得重疊。它們必須包含不同的節點或在不同的時間運作。如果在建立預約時未指定特定節點，Slurm 將自動選擇節點以避免重疊並確保所選節點在預約開始時可用。

可透過在預約上加入 **OVERLAP** 旗標來改變此行為。這可用於確保在群組預約中為特定使用者提供資源的可用性。例如，使用者 alan 和 brenda 有 10 節點 60 分鐘的預約，我們可能想要在時間段的前 30 分鐘為 brenda 保留其中的 4 個節點。在這種情況下，建立一個重疊預約（總共兩個預約）可提供更簡單的設定和使用方式。若不使用此旗標，則需要建立三個單獨的預約，且使用者在最後 30 分鐘需要在兩個不同的預約名稱之間選擇：

1. 為 alan 和 brenda 提供持續完整 60 分鐘的六節點預約
2. 為 brenda 提供前 30 分鐘的四節點預約
3. 為 alan 和 brenda 提供最後 30 分鐘的四節點預約

如果在建立預約時使用 **OVERLAP** 旗標，可以在預約中建立預約，再在第三個預約中建立。請注意，具有 **OVERLAP** 旗標的預約不會被同樣具有 **OVERLAP** 旗標的後續預約移除資源，因此更多重疊預約只是允許以不同的預約名稱存取相同資源。

### 維護預約

為了便於系統維護，您可以建立具有 **MAINT** 旗標的預約，與現有預約重疊。此旗標相當於 **OVERLAP** 旗標，但有以下區別：

- 保留的節點將顯示為 **maint** 狀態
- 保留的節點不會作為其他預約的替換節點
- 叢集使用報告中此時間會顯示為 **PlannedDown**

例如，使用者 alan 和 brenda 可能每天從中午到下午 1 點有一些節點的預約。如果所有節點從下午 12:30 開始有維護預約，他們在預約中可能啟動的唯一作業必須在維護預約開始的 12:30 前完成。

### 循環預約

Slurm 允許預約按預定義的間隔重複執行。預約將保持相同的特性，並自動在 **active**（啟用中）和 **inactive**（未來）狀態之間切換。建立預約時指定以下旗標之一即可實現：**HOURLY**、**DAILY**、**WEEKLY**、**WEEKDAY** 或 **WEEKEND**。前三者直接代表重複間隔，後兩者代表預約應重複的日期類型（工作日或週末）。

**注意**：當循環預約嘗試替換節點時，它會考量資源的未來可用性，僅保留在重疊預約中或該時間點未被保留的資源。

### 浮動時間預約

Slurm 可用於建立開始時間保持在未來固定時間段的進階預約。這些預約不是為了執行作業，而是為了防止在特定節點上啟動長時間執行的作業。該節點可能被置於 DRAINING 狀態以防止在那裡啟動**任何**新作業。或者，可以在節點上放置進階預約以防止超過某些特定時間限制的作業被啟動。使用者嘗試使用具有浮動開始時間的預約將被拒絕。當準備好執行維護時，將節點置於 DRAINING 狀態並刪除先前建立的進階預約。

透過使用 **TIME_FLOAT** 旗標值和相對於當前時間的開始時間（使用關鍵字 **now**）建立預約。預約持續時間通常應該是相對於典型作業執行時間較大的值，以免對回填排程決策產生不利影響。或者，預約可以有特定的結束時間，在這種情況下，預約的開始時間將隨時間增加直到達到預約的結束時間。當當前時間超過預約結束時間時，預約將被清除。在以下範例中，節點 tux8 被阻止啟動任何超過 60 分鐘時間限制的作業。此預約的持續時間為 100（分鐘）。

```bash
$ scontrol create reservation user=operator nodes=tux8 \
  starttime=now+60minutes duration=100 flags=time_float
```

### 替換已分配資源的預約

預設情況下，預約中處於 DOWN 或 DRAINED 狀態的節點將被替換，但分配給作業的節點不會。可以使用 **REPLACE_DOWN** 旗標明確請求此行為。

但是，您可以指示 Slurm 也用新的閒置節點替換分配給作業的節點。這是使用 **REPLACE** 旗標完成的，如以下範例所示。這樣做的效果是始終維護一個恆定大小的資源池。此選項不支援指定跨越多個節點的核心而非完整節點的預約。（例如，節點「tux1」上的 1 核心預約如果節點「tux1」停機將被移動，但包含節點「tux1」上 2 個核心和「tux2」上 3 個核心的預約如果「tux1」停機將不會被移動。）

```bash
$ scontrol create reservation starttime=now duration=60 \
  users=foo nodecnt=2 flags=replace
Reservation created: foo_82

$ scontrol show res
ReservationName=foo_82 StartTime=2014-11-20T16:21:11
   EndTime=2014-11-20T17:21:11 Duration=01:00:00
   Nodes=tux[0-1] NodeCnt=2 CoreCnt=12 Features=(null)
   PartitionName=debug Flags=REPLACE
   Users=jette Accounts=(null) Licenses=(null) State=ACTIVE

$ sbatch -n4 --reservation=foo_82 tmp
Submitted batch job 97

$ scontrol show res
ReservationName=foo_82 StartTime=2014-11-20T16:21:11
   EndTime=2014-11-20T17:21:11 Duration=01:00:00
   Nodes=tux[1-2] NodeCnt=2 CoreCnt=12 Features=(null)
   PartitionName=debug Flags=REPLACE
   Users=jette Accounts=(null) Licenses=(null) State=ACTIVE

$ sbatch -n4 --reservation=foo_82 tmp
Submitted batch job 98

$ scontrol show res
ReservationName=foo_82 StartTime=2014-11-20T16:21:11
   EndTime=2014-11-20T17:21:11 Duration=01:00:00
   Nodes=tux[2-3] NodeCnt=2 CoreCnt=12 Features=(null)
   PartitionName=debug Flags=REPLACE
   Users=jette Accounts=(null) Licenses=(null) State=ACTIVE

$ squeue
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST(REASON)
   97     debug   tmp   foo  R  0:09      1 tux0
   98     debug   tmp   foo  R  0:07      1 tux1
```

### FLEX 預約

預設情況下，在預約中執行的作業必須符合保留資源的時間和大小限制。使用 **FLEX** 旗標，作業可以在預約開始前啟動或在預約結束後繼續。它們還可以在需要且可用時使用保留的節點以及額外的節點。

請求預約的作業的預設行為是它們必須能夠在該預約的範圍（時間和空間）內執行。以下範例顯示 **FLEX** 旗標允許作業在預約開始前執行、在預約結束後執行，以及在預約之外的節點上執行。

```bash
$ scontrol create reservation user=user1 nodes=node01 starttime=now+10minutes duration=10 flags=flex
Reservation created: user1_831

$ sbatch -wnode0[1-2] -t30:00 --reservation=user1_831 test.job
Submitted batch job 57996

$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             57996     debug sleepjob    user1  R       0:08      2 node[01-02]
```

### 磁性預約

預約的預設行為是作業必須請求預約才能在其中執行。**MAGNETIC** 旗標允許您建立一個允許作業在其中執行的預約，而無需它們指定預約的名稱。預約只會「吸引」符合存取控制要求的作業。

**注意**：磁性預約無法「吸引」異質作業 - 異質作業只有在明確請求預約時才會在磁性預約中執行。

以下範例顯示在 node05 上建立的預約。被指定為能夠存取預約的使用者然後提交作業，作業在保留的節點上啟動。

```bash
$ scontrol create reservation user=user1 nodes=node05 starttime=now duration=10 flags=magnetic
Reservation created: user1_850

$ scontrol show res
ReservationName=user1_850 StartTime=2020-07-29T13:44:13 EndTime=2020-07-29T13:54:13 Duration=00:10:00
   Nodes=node05 NodeCnt=1 CoreCnt=12 Features=(null) PartitionName=(null) Flags=SPEC_NODES,MAGNETIC
   TRES=cpu=12
   Users=user1 Accounts=(null) Licenses=(null) State=ACTIVE BurstBuffer=(null)
   MaxStartDelay=(null)

$ sbatch -N1 -t5:00 test.job
Submitted batch job 62297

$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             62297     debug sleepjob    user1  R       0:04      1 node05
```

### 最後作業完成後清除預約

預約可以在最後一個關聯作業完成後自動清除。這是透過使用「purge_comp」旗標來完成的。一旦預約被建立，它必須在開始時間的 5 分鐘內被填充，否則它將在任何作業執行之前被清除。

### 預約會計

在預約中執行的作業使用適當的使用者和帳號進行會計。如果預約中的資源未被使用，這些資源將被會計為由與預約關聯的所有使用者或帳號平均使用（例如，如果兩個使用者有資格使用預約但都沒有使用，每個使用者將被報告為使用了一半的保留資源）。

### 前置和後置腳本

Slurm 支援預約前置腳本 (prolog) 和後置腳本 (epilog)。它們可以使用 slurm.conf 檔案中的 **ResvProlog** 和 **ResvEpilog** 配置參數進行配置。這些腳本可用於取消作業、修改分割區配置等。

### 未來工作

在具有幫派排程的分割區中進行的預約在考慮作業啟動時假設最高級別的時間分片而不是實際級別。這將阻止啟動一些在較少作業進行時間分片的情況下會在預約前完成執行的作業。

---

## Explanation

### 核心概念

1. **預約 (Reservation)**：預先保留的叢集資源，在特定時間段內為特定使用者或帳號獨佔使用
2. **資源類型**：可保留核心、節點、授權（licenses）、突發緩衝區（burst buffers）
3. **存取控制**：可設定允許或排除特定使用者、帳號、QOS、分割區

### 常用旗標

| 旗標 | 說明 |
|------|------|
| 旗標 | 說明 |
|------|------|
| `maint` | 維護預約，與 OVERLAP 相同但節點顯示 maint 狀態、計費為 PlannedDown |
| `overlap` | 允許與既有預約重疊 |
| `ignore_jobs` | 建立預約時忽略執行中的作業 |
| `hourly` | 每小時循環重複的預約 |
| `daily` | 每日循環重複的預約 |
| `weekly` | 每週循環重複的預約 |
| `weekday` | 僅在工作日（週一至週五）重複的預約 |
| `weekend` | 僅在週末（週六、週日）重複的預約 |
| `flex` | 作業可在預約時間外執行 |
| `magnetic` | 自動吸引符合條件的作業 |
| `replace` | 用閒置節點替換已分配的節點 |
| `time_float` | 開始時間持續浮動在未來 |
| `license_only` | 只預約授權，不預約節點 |
| `any_nodes` | 預約不需要特定節點 |
| `purge_comp` | 最後作業完成後自動清除預約 |

---

## Practical Example

### 場景：為研究團隊建立每週例行計算預約

研究團隊需要每週三下午 2 點到 6 點使用 8 個 GPU 節點進行機器學習訓練。

```bash
# 1. 建立每週重複的預約
$ scontrol create reservation \
    reservationname=ml_training \
    user=researcher1,researcher2,researcher3 \
    starttime=2024-01-10T14:00:00 \
    duration=240 \
    nodecnt=8 \
    partition=gpu \
    flags=weekly
# reservationname=  指定預約名稱為 ml_training
# user=             允許這三位研究人員使用預約
# starttime=        設定開始時間為週三下午 2 點
# duration=240      預約持續 240 分鐘（4 小時）
# nodecnt=8         預約 8 個節點
# partition=gpu     從 gpu 分割區選擇節點
# flags=weekly      每週重複此預約

# 2. 查看預約詳情
$ scontrol show reservation ml_training
ReservationName=ml_training StartTime=2024-01-10T14:00:00
   EndTime=2024-01-10T18:00:00 Duration=04:00:00
   Nodes=gpu[01-08] NodeCnt=8 CoreCnt=256
   Features=(null) PartitionName=gpu
   Flags=WEEKLY Licenses=(null)
   Users=researcher1,researcher2,researcher3 Accounts=(null)

# 3. 研究人員提交作業到預約
$ sbatch --reservation=ml_training -N4 --gres=gpu:4 train_model.sh
sbatch: Submitted batch job 12345

# 4. 需要時修改預約（例如延長時間）
$ scontrol update ReservationName=ml_training duration=300
Reservation updated.

# 5. 取消預約（如果不再需要）
$ scontrol delete ReservationName=ml_training
```

### 場景：為系統維護建立全叢集預約

```bash
# 建立下週一凌晨 2 點的維護視窗，持續 4 小時
$ scontrol create reservation \
    starttime=2024-01-15T02:00:00 \
    duration=240 \
    user=root \
    nodes=ALL \
    flags=maint,ignore_jobs
# nodes=ALL         保留所有節點
# flags=maint       標記為維護用途（會計系統會記錄）
# flags=ignore_jobs 忽略目前執行中的作業
```

---

## Common Mistakes & Tips

### 常見錯誤

1. **忘記指定預約名稱**
   ```bash
   # 錯誤：作業沒有指定預約
   $ sbatch -N4 my_job.sh
   # 即使有預約，作業也不會使用預留資源

   # 正確：明確指定預約
   $ sbatch --reservation=my_resv -N4 my_job.sh
   ```

2. **作業超出預約時間限制**
   ```bash
   # 錯誤：作業時間超過預約持續時間
   $ sbatch --reservation=my_resv -t 5:00:00 job.sh  # 預約只有 2 小時
   # 作業會被拒絕或在預約結束時被取消

   # 正確：確保作業時間在預約範圍內
   $ sbatch --reservation=my_resv -t 1:30:00 job.sh
   ```

3. **預約核心時使用 select/linear**
   ```bash
   # 錯誤：使用核心預約時系統配置了 select/linear
   # 核心預約在 select/linear 下不可用

   # 解決：使用節點預約，或更改為 select/cons_tres
   ```

4. **重疊預約未使用正確旗標**
   ```bash
   # 錯誤：建立重疊預約但沒有 maint 或 overlap 旗標
   # 會導致預約建立失敗

   # 正確：使用 overlap 旗標
   $ scontrol create reservation ... flags=overlap
   ```

### 實用建議

- **提前建立預約**：預約會在建立時立即分配節點，讓使用者有時間暫存資料
- **使用磁性預約**：對於常規使用者，磁性預約更方便，無需每次指定預約名稱
- **監控預約使用率**：未使用的預約資源仍會被計入帳號的使用量
- **配合 ResvOverRun**：在 slurm.conf 設定 ResvOverRun 允許作業在預約結束後繼續執行一段時間
- **使用 FLEX 增加彈性**：需要更大彈性時，FLEX 旗標允許作業跨越預約邊界

---

## Quick Reference

| 指令 | 說明 |
|------|------|
| `scontrol create reservation ...` | 建立新預約 |
| `scontrol show reservation` | 顯示所有預約 |
| `scontrol show res <name>` | 顯示特定預約詳情 |
| `scontrol update ReservationName=<name> ...` | 修改預約 |
| `scontrol delete ReservationName=<name>` | 刪除預約 |
| `sbatch --reservation=<name>` | 提交作業到預約 |
| `squeue --reservation=<name>` | 查看預約中的作業 |

### 建立預約常用參數

| 參數 | 說明 |
|------|------|
| `starttime=` | 開始時間（now、noon、YYYY-MM-DDTHH:MM:SS） |
| `duration=` | 持續時間（分鐘）或 infinite |
| `nodecnt=` | 節點數量 |
| `nodes=` | 特定節點（ALL 表示所有） |
| `corecnt=` | 核心數量 |
| `user=` | 允許的使用者（-user 表示排除） |
| `account=` | 允許的帳號 |
| `partition=` | 分割區 |
| `flags=` | 旗標（maint、daily、flex 等） |
| `licenses=` | 授權名稱:數量 |
| `burstbuffer=` | 突發緩衝區大小 |
| `tres=` | TRES 資源規格 |
