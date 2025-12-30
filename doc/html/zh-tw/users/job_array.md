# Slurm 作業陣列支援

---

## TL;DR

作業陣列（Job Array）是一種快速提交和管理大量相似作業的機制。使用 `sbatch --array=0-31` 可以一次提交 32 個作業。每個任務透過 `SLURM_ARRAY_TASK_ID` 環境變數識別。可用 `%` 限制同時執行數量（如 `--array=0-100%10`）。支援相依性設定、檔名模式替換（`%A`、`%a`）等進階功能。

---

## Translation（翻譯）

### 概述

作業陣列（Job Array）提供了一種快速且輕鬆地提交和管理相似作業集合的機制，適用於遵循常見作業模式的重複性工作負載。這大大提高了整體效能，因為包含數百萬個任務的作業陣列可以在毫秒內提交（受限於配置的大小限制），而且排程器可以快速識別何時沒有更多陣列任務符合啟動條件。

所有作業必須具有相同的初始選項（例如大小、時間限制），但是可以在作業開始執行後使用 `scontrol` 指令變更某些選項，需指定陣列的 *JobID* 或個別的 *ArrayJobID*。

```bash
$ scontrol update job=101 ...
$ scontrol update job=101_1 ...
```

作業陣列僅支援批次作業（batch jobs），陣列索引值使用 `sbatch` 指令的 `--array` 或 `-a` 選項指定。選項參數可以是特定的陣列索引值、索引值範圍，以及可選的步進值，如下列範例所示。

請注意，最小索引值為零，最大值為 Slurm 配置參數 *MaxArraySize* 減一。屬於作業陣列一部分的作業將設定環境變數 `SLURM_ARRAY_TASK_ID` 為其陣列索引值。

```bash
# 提交索引值在 0 到 31 之間的作業陣列
$ sbatch --array=0-31    -N1 tmp

# 提交索引值為 1, 3, 5 和 7 的作業陣列
$ sbatch --array=1,3,5,7 -N1 tmp

# 提交索引值在 1 到 7 之間、步進值為 2 的作業陣列
# （即 1, 3, 5 和 7）
$ sbatch --array=1-7:2   -N1 tmp
```

可以使用 `%` 分隔符號指定作業陣列中同時執行任務的最大數量。例如 `--array=0-15%4` 將此作業陣列中同時執行的任務數量限制為 4 個。

**注意**：作業陣列任務仍然像普通作業一樣運作，包括作業相關限制的執行（例如 **MaxJobs**、**MaxSubmitJobs**）。

### 作業 ID 和環境變數

作業陣列將設定額外的環境變數：

| 環境變數 | 說明 |
|----------|------|
| `SLURM_ARRAY_JOB_ID` | 陣列第一個作業的 ID |
| `SLURM_ARRAY_TASK_ID` | 作業陣列索引值 |
| `SLURM_ARRAY_TASK_COUNT` | 作業陣列中的任務數量 |
| `SLURM_ARRAY_TASK_MAX` | 最高的作業陣列索引值 |
| `SLURM_ARRAY_TASK_MIN` | 最低的作業陣列索引值 |

在正常情況下，陣列作業的第一個任務將作為陣列其餘部分的預留位置，導致它最後執行。因此，具有最低 `SLURM_JOB_ID` 的任務將具有最高的 `SLURM_ARRAY_TASK_ID`。

例如這樣的作業提交：
```bash
sbatch --array=1-3 -N1 tmp
```

將產生一個包含三個作業的作業陣列。如果 sbatch 指令回應：
```
Submitted batch job 36
```

則環境變數將設定如下：

```bash
# 第一個任務（作業 ID 36）
SLURM_JOB_ID=36
SLURM_ARRAY_JOB_ID=36
SLURM_ARRAY_TASK_ID=3
SLURM_ARRAY_TASK_COUNT=3
SLURM_ARRAY_TASK_MAX=3
SLURM_ARRAY_TASK_MIN=1

# 第二個任務（作業 ID 37）
SLURM_JOB_ID=37
SLURM_ARRAY_JOB_ID=36
SLURM_ARRAY_TASK_ID=1
SLURM_ARRAY_TASK_COUNT=3
SLURM_ARRAY_TASK_MAX=3
SLURM_ARRAY_TASK_MIN=1

# 第三個任務（作業 ID 38）
SLURM_JOB_ID=38
SLURM_ARRAY_JOB_ID=36
SLURM_ARRAY_TASK_ID=2
SLURM_ARRAY_TASK_COUNT=3
SLURM_ARRAY_TASK_MAX=3
SLURM_ARRAY_TASK_MIN=1
```

上述任務的順序不保證。例如，當任務被重新排入佇列時，可能會出現個別任務不按順序建立的情況。

所有 Slurm 指令和 API 都能識別 `SLURM_JOB_ID` 值。大多數指令也能識別以底線分隔的 `SLURM_ARRAY_JOB_ID` 加 `SLURM_ARRAY_TASK_ID` 值來識別作業陣列的元素。使用上述範例，「37」或「36_1」是識別作業 36 的第二個陣列元素的等效方式。

### 檔案名稱

有兩個額外的選項可用於指定作業的 stdin、stdout 和 stderr 檔案名稱：

| 模式 | 替換為 |
|------|--------|
| `%A` | `SLURM_ARRAY_JOB_ID` 的值 |
| `%a` | `SLURM_ARRAY_TASK_ID` 的值 |

作業陣列的預設輸出檔案格式為 `slurm-%A_%a.out`。

明確使用格式的範例：
```bash
sbatch -o slurm-%A_%a.out --array=1-3 -N1 tmp
```

這將產生名為 `slurm-36_1.out`、`slurm-36_2.out` 和 `slurm-36_3.out` 的輸出檔案。

如果這些檔案名稱選項在非作業陣列的情況下使用，則 `%A` 將被替換為當前作業 ID，`%a` 將被替換為 4,294,967,294（相當於 0xfffffffe 或 NO_VAL）。

### Scancel 指令使用

如果將作業陣列的作業 ID 作為 scancel 指令的輸入，則該作業陣列的所有元素都將被取消。或者，可以指定陣列 ID（可選使用正規表示式）來取消作業。

```bash
# 取消作業陣列 20 的陣列 ID 1 到 3
$ scancel 20_[1-3]

# 取消作業陣列 20 的陣列 ID 4 和 5
$ scancel 20_4 20_5

# 取消作業陣列 20 的所有元素
$ scancel 20

# 取消當前作業或作業陣列元素（如果是作業陣列）
if [[ -z $SLURM_ARRAY_JOB_ID ]]; then
  scancel $SLURM_JOB_ID
else
  scancel ${SLURM_ARRAY_JOB_ID}_${SLURM_ARRAY_TASK_ID}
fi
```

### Squeue 指令使用

當作業陣列提交到 Slurm 時，只會建立一個作業記錄。只有當作業陣列中的任務狀態改變時（通常是當任務被分配資源或其狀態透過 scontrol 指令修改時），才會建立額外的作業記錄。

預設情況下，squeue 指令會在一行上報告與單一作業記錄相關的所有任務，並使用正規表示式來指示「array_task_id」值：

```bash
$ squeue
 JOBID     PARTITION  NAME  USER  ST  TIME  NODES NODELIST(REASON)
1080_[5-1024]  debug   tmp   mac  PD  0:00      1 (Resources)
1080_1         debug   tmp   mac   R  0:17      1 tux0
1080_2         debug   tmp   mac   R  0:16      1 tux1
1080_3         debug   tmp   mac   R  0:03      1 tux2
1080_4         debug   tmp   mac   R  0:03      1 tux3
```

squeue 指令新增了 `--array` 或 `-r` 選項，可以每行列印一個作業陣列元素：

```bash
$ squeue -r
 JOBID PARTITION  NAME  USER  ST  TIME  NODES NODELIST(REASON)
1082_3     debug   tmp   mac  PD  0:00      1 (Resources)
1082_4     debug   tmp   mac  PD  0:00      1 (Priority)
  1080     debug   tmp   mac   R  0:17      1 tux0
  1081     debug   tmp   mac   R  0:16      1 tux1
1082_1     debug   tmp   mac   R  0:03      1 tux2
1082_2     debug   tmp   mac   R  0:03      1 tux3
```

環境變數 `SQUEUE_ARRAY` 相當於在 squeue 命令列包含 `--array` 選項。

squeue 的 `--step/-s` 和 `--job/-j` 選項可以接受相同格式的作業或步驟規格：

```bash
$ squeue -j 1234_2,1234_3
...
$ squeue -s 1234_2.0,1234_3.0
...
```

squeue 新增了兩個額外的作業輸出格式欄位選項：
- **%F**：列印 array_job_id 值
- **%K**：列印 array_task_id 值

### Scontrol 指令使用

使用 `scontrol show job` 選項會顯示與作業陣列支援相關的兩個新欄位：
- **JobID**：作業的唯一識別碼
- **ArrayJobID**：作業陣列第一個元素的 JobID
- **ArrayTaskID**：此特定項目的陣列索引

如果作業不是作業陣列的一部分，則不會顯示這兩個欄位。

scontrol 指令如果指定的作業 ID 是 ArrayJobID，將對作業陣列的所有元素進行操作。可以使用 `ArrayJobID_ArrayTaskID` 修改個別作業陣列任務：

```bash
$ sbatch --array=1-4 -J array ./sleepme 86400
Submitted batch job 21845

$ squeue
 JOBID   PARTITION     NAME     USER  ST  TIME NODES NODELIST
 21845_1    canopo    array    david  R  0:13  1     dario
 21845_2    canopo    array    david  R  0:13  1     dario
 21845_3    canopo    array    david  R  0:13  1     dario
 21845_4    canopo    array    david  R  0:13  1     dario

$ scontrol update JobID=21845_2 name=arturo
$ squeue
 JOBID   PARTITION     NAME     USER  ST   TIME  NODES NODELIST
 21845_1    canopo    array    david  R   17:03   1    dario
 21845_2    canopo   arturo    david  R   17:03   1    dario
 21845_3    canopo    array    david  R   17:03   1    dario
 21845_4    canopo    array    david  R   17:03   1    dario
```

scontrol 的 hold、holdu、release、requeue、requeuehold、suspend 和 resume 指令也可以對作業陣列的所有元素或個別元素進行操作。

### 作業相依性

依賴於整個作業陣列的作業應指定其依賴於 ArrayJobID。由於每個陣列元素可以有不同的退出代碼，*afterok* 和 *afternotok* 子句的解釋將基於作業陣列中任何任務的最高退出代碼。

當作業相依性指定作業陣列的作業 ID 時：

| 相依性類型 | 滿足條件 |
|------------|----------|
| `after` | 作業陣列中的所有任務開始後 |
| `afterany` | 作業陣列中的所有任務完成後 |
| `aftercorr` | 指定作業中對應的任務 ID 成功完成後（以退出代碼 0 執行完畢） |
| `afterok` | 作業陣列中的所有任務成功完成後 |
| `afternotok` | 作業陣列中的所有任務完成，且至少有一個任務未成功完成後 |

使用範例：
```bash
# 等待特定作業陣列元素
sbatch --depend=after:123_4 my.job
sbatch --depend=afterok:123_4:123_8 my.job2

# 等待整個作業陣列完成
sbatch --depend=afterany:123 my.job

# 等待對應的作業陣列元素
sbatch --depend=aftercorr:123 my.job

# 等待整個作業陣列成功完成
sbatch --depend=afterok:123 my.job

# 等待整個作業陣列完成且至少有一個任務失敗
sbatch --depend=afternotok:123 my.job
```

### 其他指令使用

以下 Slurm 指令目前不識別作業陣列，其使用需要使用 Slurm 作業 ID（每個陣列元素都是唯一的）：sbcast、sprio、sreport、sshare 和 sstat。

sacct、sattach 和 strigger 指令已被修改為允許指定作業 ID 或作業陣列元素。

sview 指令已被修改為允許顯示作業的 ArrayJobId 和 ArrayTaskId 欄位。如果作業不是作業陣列的一部分，這兩個欄位都會顯示為「N/A」。

### 系統管理

新增了一個配置參數來控制最大作業陣列大小：**MaxArraySize**。使用者可以指定的最小索引是零，最大索引是 MaxArraySize 減一。MaxArraySize 的預設值為 1001。Slurm 支援的最大 MaxArraySize 為 4000001。

請注意 MaxArraySize 的值，因為作業陣列提供了一種讓使用者非常快速地提交大量作業的簡便方式。

sched/backfill 外掛已被修改以提高作業陣列的效能。一旦發現作業陣列的一個元素不可執行或影響待處理作業的排程，該作業陣列的其餘元素將被快速跳過。

Slurm 在提交作業陣列時建立一個單一作業記錄。只有在需要時才會建立額外的作業記錄，通常是在作業陣列的任務開始時，這提供了一個非常可擴展的機制來管理大量作業計數。

---

## Explanation（解釋）

### 什麼是作業陣列？

想像你有一個程式需要對 1000 個不同的輸入檔案執行相同的處理。傳統方式是寫一個迴圈提交 1000 個作業：

```bash
# ❌ 不好的方式：提交 1000 個獨立作業
for i in {1..1000}; do
    sbatch my_job.sh input_$i.txt
done
```

這樣做的問題：
- 需要 1000 次 sbatch 呼叫
- 排程器需要管理 1000 個獨立的作業記錄
- 效能低落且佔用系統資源

使用作業陣列：
```bash
# ✅ 好的方式：一次提交作業陣列
sbatch --array=1-1000 my_job.sh
```

### 作業陣列的運作原理

```
提交作業陣列
      │
      ▼
┌─────────────────────────────────────────────────┐
│              作業陣列 (Job Array)                │
│                 ArrayJobID = 100                │
│                                                 │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     ┌─────┐  │
│  │100_1│ │100_2│ │100_3│ │100_4│ ... │100_N│  │
│  │     │ │     │ │     │ │     │     │     │  │
│  │ID=1 │ │ID=2 │ │ID=3 │ │ID=4 │     │ID=N │  │
│  └─────┘ └─────┘ └─────┘ └─────┘     └─────┘  │
│                                                 │
│  每個任務獲得獨立的 SLURM_ARRAY_TASK_ID          │
└─────────────────────────────────────────────────┘
```

### 索引值的靈活性

```bash
# 連續範圍
--array=0-99          # 0, 1, 2, ..., 99

# 指定的值
--array=1,5,10,20     # 只有 1, 5, 10, 20

# 帶步進的範圍
--array=0-100:10      # 0, 10, 20, 30, ..., 100

# 限制同時執行數量
--array=0-999%50      # 最多同時執行 50 個
```

---

## Practical Example（實用範例）

### 範例 1：基本作業陣列

```bash
#!/bin/bash
#SBATCH --job-name=array_test       # 作業名稱
#SBATCH --array=1-10                # 陣列索引 1 到 10
#SBATCH --output=logs/job_%A_%a.out # 輸出檔案（%A=陣列ID, %a=任務ID）
#SBATCH --error=logs/job_%A_%a.err  # 錯誤檔案
#SBATCH --time=00:10:00             # 每個任務最大時間
#SBATCH --ntasks=1                  # 每個任務使用 1 個 CPU
#SBATCH --mem=1G                    # 每個任務使用 1GB 記憶體

# 顯示任務資訊
echo "陣列作業 ID: $SLURM_ARRAY_JOB_ID"
echo "任務 ID: $SLURM_ARRAY_TASK_ID"
echo "任務總數: $SLURM_ARRAY_TASK_COUNT"

# 使用任務 ID 處理對應的輸入檔案
INPUT_FILE="data/input_${SLURM_ARRAY_TASK_ID}.txt"
OUTPUT_FILE="results/output_${SLURM_ARRAY_TASK_ID}.txt"

# 執行處理
./my_program $INPUT_FILE > $OUTPUT_FILE
```

**逐行說明：**
- `#SBATCH --array=1-10`：建立 10 個任務，索引 1-10
- `%A_%a`：檔名模式，會被替換為實際的陣列 ID 和任務 ID
- `$SLURM_ARRAY_TASK_ID`：每個任務的唯一識別碼，用來處理不同的輸入

### 範例 2：限制同時執行數量

```bash
#!/bin/bash
#SBATCH --job-name=limited_array
#SBATCH --array=1-100%10            # 100 個任務，但最多同時執行 10 個
#SBATCH --output=logs/%A_%a.out
#SBATCH --time=01:00:00

echo "正在處理任務 $SLURM_ARRAY_TASK_ID (共 $SLURM_ARRAY_TASK_COUNT 個)"

# 你的程式碼
sleep 60
```

**說明：**
- `%10` 表示最多同時執行 10 個任務
- 適用於：資源有限、避免過載檔案系統、限制授權軟體並發數

### 範例 3：使用陣列索引作為參數

```bash
#!/bin/bash
#SBATCH --array=0-9

# 定義參數陣列
PARAMS=("alpha" "beta" "gamma" "delta" "epsilon"
        "zeta" "eta" "theta" "iota" "kappa")

# 取得對應此任務的參數
PARAM=${PARAMS[$SLURM_ARRAY_TASK_ID]}

echo "任務 $SLURM_ARRAY_TASK_ID 使用參數: $PARAM"
./simulation --param=$PARAM
```

### 範例 4：從檔案讀取參數

```bash
#!/bin/bash
#SBATCH --array=1-100

# 參數檔案格式：每行一組參數
# params.txt 內容範例：
# input1.txt 0.5 100
# input2.txt 0.7 200
# ...

# 讀取對應行
LINE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" params.txt)

# 解析參數
INPUT=$(echo $LINE | awk '{print $1}')
THRESHOLD=$(echo $LINE | awk '{print $2}')
ITERATIONS=$(echo $LINE | awk '{print $3}')

echo "處理: $INPUT, 閾值=$THRESHOLD, 迭代=$ITERATIONS"
./process --input=$INPUT --threshold=$THRESHOLD --iter=$ITERATIONS
```

### 範例 5：作業相依性

```bash
# 提交第一個作業陣列
JOB1=$(sbatch --array=1-10 --parsable preprocess.sh)
echo "前處理作業: $JOB1"

# 提交依賴於整個陣列完成的作業
JOB2=$(sbatch --depend=afterok:$JOB1 merge_results.sh)
echo "合併作業: $JOB2 (等待 $JOB1 完成)"

# 提交具有對應相依性的陣列（任務 1 等待任務 1）
JOB3=$(sbatch --array=1-10 --depend=aftercorr:$JOB1 postprocess.sh)
echo "後處理作業: $JOB3 (對應相依於 $JOB1)"
```

### 範例 6：取消和監控

```bash
# 查看作業陣列狀態
squeue -j 12345

# 查看每個任務一行
squeue -j 12345 -r

# 取消特定任務
scancel 12345_5

# 取消一個範圍的任務
scancel 12345_[10-20]

# 取消整個陣列
scancel 12345

# 暫停特定任務
scontrol suspend 12345_3

# 恢復執行
scontrol resume 12345_3
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| 輸出檔名未使用 `%a` | 所有任務輸出到同一檔案，互相覆蓋 | 使用 `-o output_%A_%a.out` |
| 未建立輸出目錄 | 作業失敗因為目錄不存在 | 提交前先 `mkdir -p logs` |
| 索引超出陣列範圍 | 嘗試存取不存在的參數 | 確保參數陣列/檔案行數足夠 |
| 同時執行太多任務 | 檔案系統或網路過載 | 使用 `%N` 限制同時執行數量 |
| 硬編碼陣列大小 | 與參數檔案行數不符 | 動態計算：`wc -l < params.txt` |

### ✅ 實用技巧

1. **動態設定陣列大小**
   ```bash
   # 根據輸入檔案數量
   NUM_FILES=$(ls input/*.txt | wc -l)
   sbatch --array=1-$NUM_FILES job.sh
   ```

2. **使用帶步進的陣列分批處理**
   ```bash
   # 每個任務處理 10 個項目
   #SBATCH --array=0-990:10

   START=$SLURM_ARRAY_TASK_ID
   END=$((START + 9))
   for i in $(seq $START $END); do
       process_item $i
   done
   ```

3. **記錄完整的任務資訊**
   ```bash
   echo "========================================="
   echo "作業陣列 ID: $SLURM_ARRAY_JOB_ID"
   echo "任務 ID: $SLURM_ARRAY_TASK_ID"
   echo "任務範圍: $SLURM_ARRAY_TASK_MIN - $SLURM_ARRAY_TASK_MAX"
   echo "主機名稱: $(hostname)"
   echo "開始時間: $(date)"
   echo "========================================="
   ```

4. **處理失敗的任務**
   ```bash
   # 查看失敗的任務
   sacct -j 12345 --format=JobID,State,ExitCode | grep -E "FAILED|TIMEOUT"

   # 只重新執行失敗的任務
   sbatch --array=5,12,37 job.sh  # 假設這些任務失敗
   ```

5. **建立輸出目錄結構**
   ```bash
   # 在腳本開頭
   mkdir -p results/${SLURM_ARRAY_JOB_ID}
   OUTPUT_DIR="results/${SLURM_ARRAY_JOB_ID}/${SLURM_ARRAY_TASK_ID}"
   mkdir -p $OUTPUT_DIR
   ```

---

## Quick Reference（快速參考）

### 作業陣列語法

| 語法 | 說明 | 範例 |
|------|------|------|
| `--array=N-M` | 連續範圍 | `--array=1-100` |
| `--array=N,M,O` | 指定值 | `--array=1,5,10` |
| `--array=N-M:S` | 帶步進的範圍 | `--array=0-100:10` |
| `--array=N-M%L` | 限制同時執行 | `--array=1-1000%50` |

### 環境變數

| 變數 | 說明 |
|------|------|
| `SLURM_ARRAY_JOB_ID` | 陣列的主作業 ID |
| `SLURM_ARRAY_TASK_ID` | 當前任務的索引 |
| `SLURM_ARRAY_TASK_COUNT` | 陣列中的任務總數 |
| `SLURM_ARRAY_TASK_MAX` | 最大索引值 |
| `SLURM_ARRAY_TASK_MIN` | 最小索引值 |

### 檔名模式

| 模式 | 替換為 |
|------|--------|
| `%A` | 陣列作業 ID |
| `%a` | 任務 ID |
| `%j` | 作業 ID |
| `%x` | 作業名稱 |

### 相依性類型

| 類型 | 條件 |
|------|------|
| `after:jobid` | 作業開始後 |
| `afterany:jobid` | 作業完成後（任何狀態） |
| `afterok:jobid` | 作業成功完成後 |
| `afternotok:jobid` | 作業失敗後 |
| `aftercorr:jobid` | 對應任務 ID 完成後 |

### 管理指令

| 指令 | 功能 |
|------|------|
| `squeue -j JOBID` | 查看陣列狀態 |
| `squeue -j JOBID -r` | 每個任務一行顯示 |
| `scancel JOBID` | 取消整個陣列 |
| `scancel JOBID_N` | 取消特定任務 |
| `scancel JOBID_[N-M]` | 取消任務範圍 |
| `scontrol show job JOBID_N` | 顯示任務詳情 |
| `scontrol update job=JOBID_N` | 更新任務屬性 |
