# Slurm 突發緩衝指南 (Burst Buffer Guide)

## TL;DR

突發緩衝 (Burst Buffer) 是高速儲存資源，Slurm 可在作業生命週期的不同階段（提交、stage-in、pre-run、stage-out、teardown）呼叫腳本進行管理。支援兩種外掛程式：datawarp（Cray DataWarp）和 lua（自訂 Lua 腳本）。透過 `BurstBufferType` 設定外掛程式，作業使用 `#DW` 或 `#BB_LUA` 指令請求緩衝資源。

---

## 翻譯

### 概觀

本指南說明如何使用 Slurm 突發緩衝外掛程式，並解釋這些外掛程式的運作方式，以提供最佳使用指引。

Slurm 突發緩衝外掛程式會在作業生命週期的不同時間點呼叫腳本：

| 階段 | 時機 | 說明 |
|------|------|------|
| **job submission** | 作業提交時 | 驗證突發緩衝指令 |
| **stage-in** | 作業等待中，預估開始時間確定後 | 資料輸入階段 |
| **pre-run** | 作業已排程但尚未開始執行 | 執行前準備 |
| **stage-out** | 作業完成或取消後，資源尚未釋放 | 資料輸出階段 |
| **teardown** | 作業完成，資源已釋放 | 清理階段 |

此腳本在 slurmctld 節點上執行。支援的外掛程式：

- **datawarp** - Cray DataWarp API
- **lua** - 自訂 Lua 腳本 API

#### Datawarp 外掛程式

提供 Cray DataWarp API 的掛勾。DataWarp 實作突發緩衝，是共享的高速儲存資源。Slurm 支援分配這些資源、stage-in 檔案、為使用這些資源的作業排程運算節點，以及 stage-out 檔案。突發緩衝也可作為作業執行期間的暫存空間，無需檔案分段。另一個典型用例是持久儲存，不與任何特定作業關聯。

#### Lua 外掛程式

提供由 Lua 腳本定義的 API 掛勾。此外掛程式旨在讓系統管理員在作業生命週期的不同時間點執行任何任務（不僅限於檔案分段）。這些任務可能包括檔案分段、節點維護，或在上述五個作業狀態期間需要執行的任何其他任務。

突發緩衝 API 僅對明確請求使用的作業呼叫。

---

### 設定（系統管理員）

#### 通用設定

| 設定項目 | 說明 |
|----------|------|
| `BurstBufferType` | 在 slurm.conf 中設定以啟用突發緩衝外掛程式，只能指定一個 |
| `DebugFlags=BurstBuffer` | 啟用詳細的突發緩衝日誌（除錯用）|
| `AccountingStorageTres` | 新增 `bb/datawarp` 或 `bb/lua` 以追蹤突發緩衝資源 |
| `burst_buffer.conf` | 突發緩衝特定設定，包括使用者權限、逾時、腳本路徑等 |

**注意**：需要安裝 JSON-C 函式庫才能建構 `burst_buffer/datawarp` 和 `burst_buffer/lua` 外掛程式。

#### Datawarp 設定

**slurm.conf：**
```
BurstBufferType=burst_buffer/datawarp
```

datawarp 外掛程式呼叫兩個腳本：

| 腳本 | 說明 | 設定位置 |
|------|------|----------|
| **dw_wlm_cli** | 執行突發緩衝功能 | burst_buffer.conf 的 GetSysState |
| **dwstat** | 取得狀態資訊 | burst_buffer.conf 的 GetSysStatus |

#### Lua 設定

**slurm.conf：**
```
BurstBufferType=burst_buffer/lua
```

lua 外掛程式呼叫單一腳本 `burst_buffer.lua`，必須位於 slurm.conf 相同目錄。

**必要函數：**
- `slurm_bb_job_process`
- `slurm_bb_pools`
- `slurm_bb_job_teardown`
- `slurm_bb_setup`
- `slurm_bb_data_in`
- `slurm_bb_test_data_in`
- `slurm_bb_real_size`
- `slurm_bb_paths`
- `slurm_bb_pre_run`
- `slurm_bb_post_run`
- `slurm_bb_data_out`
- `slurm_bb_test_data_out`
- `slurm_bb_get_status`

範本位置：`etc/burst_buffer.lua.example`

---

### Lua 實作細節

#### burst_buffer.lua 執行方式

為避免從 slurmctld 呼叫 `fork()` 影響效能，burst_buffer.lua 以兩種方式呼叫：

| 函數 | 執行位置 | 可否終止 | 說明 |
|------|----------|----------|------|
| `slurm_bb_job_process`、`slurm_bb_pools`、`slurm_bb_paths` | slurmctld 內 | 否 | 必須快速執行 |
| 其他函數 | slurmscriptd | 是 | 可長時間執行，超過逾時會被終止 |

**警告**：
- 不要在 burst_buffer.lua 中安裝信號處理程式，會導致 slurmctld 崩潰
- 最多允許 512 個 burst_buffer.lua 副本同時執行

---

### 突發緩衝資源

突發緩衝 API 可定義資源「池」(pools)，作業可從中請求一定數量的池空間。

#### 資源池運作方式

1. 如果池空間不足，作業保持等待
2. 池空間足夠時，Slurm 開始 stage-in
3. Stage-in 開始時，從池的可用空間減去作業請求的空間
4. Teardown 完成時，將作業請求的空間加回池的可用空間

#### Datawarp 資源

- 池由 `dw_wlm_cli` 定義，以 JSON 格式輸出
- 如果作業未指定池，使用 burst_buffer.conf 的 `DefaultPool`
- 如果未指定池且未定義 `DefaultPool`，作業被拒絕

#### Lua 資源

- 池是選用的，可代表任何內容
- 不使用 `DefaultPool`
- 池由 `slurm_bb_pools` 函數定義，返回 JSON 字串

**JSON 池欄位：**
- **id** - 池名稱
- **quantity** - 池空間數量
- **granularity** - 可分配的最小空間單位

---

### 作業提交命令

批次作業在腳本中使用註解指令指定突發緩衝需求。所有突發緩衝階段發生在作業生命週期的特定時間點，而非執行期間。

#### 容量規格

容量可包含後綴：
- 節點數："N"
- 1024 次方："K|KiB", "M|MiB", "G|GiB", "T|TiB", "P|PiB"
- 1000 次方："KB", "MB", "GB", "TB", "PB"

#### Datawarp 作業提交

使用 `#DW` 指令：

```bash
#!/bin/bash
#DW jobdw type=scratch capacity=1GB access_mode=striped,private pfs=/scratch
#DW stage_in type=file source=/tmp/a destination=/ss/file1
#DW stage_out type=file destination=/tmp/b source=/ss/file1
srun application.sh
```

#### Lua 作業提交

預設使用 `#BB_LUA` 指令（可在 burst_buffer.conf 用 `Directive` 選項更改）：

```bash
#!/bin/bash
#BB_LUA
srun application.sh
```

指定池和容量：
```bash
#!/bin/bash
#BB_LUA pool=pool1 capacity=1K
srun application.sh
```

---

### 持久突發緩衝（僅 Datawarp）

#### 建立和刪除指令

```bash
#BB create_persistent name=<name> capacity=<number> [access=<access>] [pool=<pool>] [type=<type>]
#BB destroy_persistent name=<name> [hurry]
```

**選項：**
- **name** - 名稱不能以數字開頭
- **capacity** - 容量大小
- **pool** - 資源池
- **access** - 存取模式（striped、private、ldbalance）
- **type** - 緩衝類型（cache、scratch）

**範例 - 建立持久緩衝：**
```bash
#!/bin/bash
#BB create_persistent name=alpha capacity=32GB access=striped type=scratch
#BB create_persistent name=beta capacity=16GB access=striped type=scratch
srun application.sh
```

**範例 - 刪除持久緩衝：**
```bash
#!/bin/bash
#BB destroy_persistent name=alpha
#BB destroy_persistent name=beta
srun application.sh
```

**零運算資源作業：**
```bash
sbatch -N0 setup_buffers.bash
```

---

### 命令列選項

#### --bb 和 --bbf 選項

| 選項 | 說明 |
|------|------|
| `--bb` | 直接指定突發緩衝指令 |
| `--bbf` | 指定包含突發緩衝指令的檔案 |

可用於 salloc、sbatch 和 srun。

**Datawarp 範例：**
```bash
srun --bb="capacity=1G access=striped type=scratch" a.out
```

**Lua 範例：**
```bash
srun --bb="#BB_LUA pool=pool1 capacity=1K"
```

---

### 符號替換

| 符號 | 說明 |
|------|------|
| %% | % |
| %A | 陣列主作業 ID |
| %a | 陣列任務 ID |
| %d | 工作目錄 |
| %j | 作業 ID |
| %u | 使用者名稱 |
| %x | 作業名稱 |
| \\ | 停止進一步處理該行 |

---

### 狀態命令

```bash
# 查看突發緩衝資訊
scontrol show burst

# 查看狀態 API（Datawarp）
scontrol show dwstat
scontrol show dwstat sessions
scontrol show dwstat configurations

# 查看狀態 API（Lua）
scontrol show bbstat
```

---

### 進階保留

突發緩衝資源可放入進階保留：

```bash
# 僅突發緩衝保留
scontrol create reservation starttime=now duration=60 \
  users=alan flags=any_nodes burstbuffer=datawarp:100G

# 含節點的保留
scontrol create reservation StartTime=noon duration=60 \
  users=brenda NodeCnt=8 BurstBuffer=datawarp:20G

# 指定池的保留
scontrol create reservation StartTime=16:00 duration=60 \
  users=joseph flags=any_nodes BurstBuffer=datawarp:pool_test:4G
```

格式：`[plugin:][pool:]#[units]`

---

### 作業相依性

| 相依性類型 | 說明 |
|------------|------|
| `--dependency=afterok:123` | 第一個作業的 stage-out 完成後才開始（如果第二個作業也用突發緩衝）|
| `--dependency=afterburstbuffer:123` | 強制等待第一個作業的 stage-out 完成 |

---

### 突發緩衝狀態與作業狀態

#### 突發緩衝狀態

| 狀態 | 說明 |
|------|------|
| pending | 等待中 |
| allocating | 正在分配（僅持久緩衝）|
| allocated | 已分配 |
| deleting | 正在刪除（僅持久緩衝）|
| deleted | 已刪除（僅持久緩衝）|
| staging-in | 資料輸入中 |
| staged-in | 資料輸入完成 |
| pre-run | 執行前準備 |
| alloc-revoke | 分配撤銷（不應發生）|
| running | 執行中 |
| suspended | 已暫停 |
| post-run | 執行後處理 |
| staging-out | 資料輸出中 |
| teardown | 清理中 |
| teardown-fail | 清理失敗 |
| complete | 完成 |

#### 典型狀態轉換

1. 作業提交 → 作業狀態：pending，緩衝狀態：pending
2. Stage-in 開始 → 作業狀態：pending (BurstBufferStageIn)，緩衝狀態：staging-in
3. Stage-in 完成 → 作業狀態：pending，緩衝狀態：staged-in
4. 作業排程 → 作業狀態：running+configuring，緩衝狀態：pre-run
5. Pre-run 完成 → 作業狀態：running，緩衝狀態：running
6. 作業完成 → 作業狀態：stage-out，緩衝狀態：staging-out
7. Stage-out 完成 → 作業狀態：complete，緩衝狀態：teardown

---

## 說明

### Datawarp vs Lua 外掛程式

| 特性 | Datawarp | Lua |
|------|----------|-----|
| 用途 | Cray DataWarp 系統 | 通用自訂任務 |
| 池 | 必須 | 選用 |
| 持久緩衝 | 支援 | 不支援 |
| 指令 | #DW | #BB_LUA（可自訂）|
| 腳本 | dw_wlm_cli, dwstat | burst_buffer.lua |
| 靈活性 | 較低 | 較高 |

### 資源池概念

```
池 (Pool)
├── 總空間 (TotalSpace)
├── 可用空間 (FreeSpace)
└── 已用空間 (UsedSpace)

作業請求空間：
1. 檢查池是否有足夠空間
2. Stage-in 開始時扣除空間
3. Teardown 完成時歸還空間
```

---

## 實務範例

### 基本 Lua 突發緩衝設定

**slurm.conf：**
```
BurstBufferType=burst_buffer/lua
AccountingStorageTres=bb/lua
```

**burst_buffer.conf：**
```
AllowUsers=user1,user2
Flags=DisablePersistent
StageInTimeout=3600
StageOutTimeout=3600
OtherTimeout=300
```

**burst_buffer.lua（簡化版）：**
```lua
function slurm_bb_job_process(job_id, uid, gid, pool, bb_size, job_script)
    slurm.log_info("Job %u processing burst buffer", job_id)
    return slurm.SUCCESS
end

function slurm_bb_pools()
    local pools = {
        {id = "pool1", quantity = 10000, granularity = 1000}
    }
    return slurm.SUCCESS, slurm.json_encode(pools)
end

-- 其他必要函數...
```

### 資料分段作業腳本

```bash
#!/bin/bash
#SBATCH --job-name=data_process
#SBATCH --nodes=4
#DW jobdw type=scratch capacity=100GB access_mode=striped pfs=/scratch
#DW stage_in type=directory source=/projects/data destination=/bb/input
#DW stage_out type=directory destination=/projects/results source=/bb/output

# 使用突發緩衝的高速儲存
cp /bb/input/* /bb/working/
srun ./process_data.sh
cp /bb/working/results/* /bb/output/
```

### 監控突發緩衝

```bash
# 查看整體狀態
scontrol show burst

# 查看特定作業的緩衝狀態
scontrol show job <jobid> | grep BurstBuffer

# 查看 Datawarp 會話
scontrol show dwstat sessions

# 查看 Datawarp 設定
scontrol show dwstat configurations
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未安裝 JSON-C 函式庫 | 建構前安裝 libjson-c-dev |
| burst_buffer.lua 位置錯誤 | 放在 slurm.conf 相同目錄 |
| Lua 函數執行太慢 | slurmctld 呼叫的函數必須快速返回 |
| 未設定逾時 | 在 burst_buffer.conf 設定合理的逾時值 |
| 忽略 stage-in 失敗 | 設定 Flags=teardownFailure 自動清理 |

### 效能考量

1. **slurmctld 效能**：
   - `slurm_bb_job_process`、`slurm_bb_pools`、`slurm_bb_paths` 必須快速執行
   - 避免在這些函數中進行 I/O 操作或網路呼叫

2. **並行限制**：
   - 最多 512 個 burst_buffer.lua 副本同時執行
   - 大量作業時注意資源競爭

3. **逾時設定**：
   ```
   StageInTimeout=86400    # Stage-in 逾時（秒）
   StageOutTimeout=86400   # Stage-out 逾時（秒）
   OtherTimeout=300        # 其他操作逾時（秒）
   ValidateTimeout=5       # 驗證逾時（秒）
   ```

### 除錯技巧

```bash
# 啟用詳細日誌
# slurm.conf
DebugFlags=BurstBuffer

# 查看突發緩衝日誌
grep -i "burst\|buffer\|bb_" /var/log/slurm/slurmctld.log

# 測試 Lua 腳本語法
lua -l burst_buffer -e "print('OK')"

# 驗證設定
scontrol show config | grep -i burst
```

---

## 快速參考

### slurm.conf 設定

```
# Datawarp 外掛程式
BurstBufferType=burst_buffer/datawarp
AccountingStorageTres=bb/datawarp

# 或 Lua 外掛程式
BurstBufferType=burst_buffer/lua
AccountingStorageTres=bb/lua

# 除錯
DebugFlags=BurstBuffer
```

### burst_buffer.conf 設定

```
AllowUsers=user1,user2
DenyUsers=user3
Flags=DisablePersistent,teardownFailure
StageInTimeout=86400
StageOutTimeout=86400
OtherTimeout=300
ValidateTimeout=5
Directive=BB_LUA
```

### 作業腳本指令

| 外掛程式 | 指令格式 |
|----------|----------|
| Datawarp | `#DW jobdw type=scratch capacity=1GB ...` |
| Datawarp 持久 | `#BB create_persistent name=xxx capacity=1GB` |
| Lua | `#BB_LUA [pool=xxx] [capacity=xxx]` |

### 常用命令

| 命令 | 功能 |
|------|------|
| `scontrol show burst` | 顯示突發緩衝狀態 |
| `scontrol show job <id>` | 顯示作業緩衝狀態 |
| `scontrol show dwstat` | 顯示 Datawarp 狀態 |
| `scontrol show bbstat` | 顯示 Lua 緩衝狀態 |
| `scancel --hurry <id>` | 跳過 stage-out 取消作業 |

### 相關文件

- [burst_buffer.conf](burst_buffer.conf.html) - 突發緩衝設定檔
- [資源限制](resource_limits.md) - TRES 限制設定
- [多因子優先順序](priority_multifactor.md) - 優先順序設定
- [異質作業](../users/heterogeneous_jobs.md) - 異質作業支援
