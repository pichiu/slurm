# Slurm 作業結束代碼 (Job Exit Codes)

## TL;DR

作業結束代碼（exit code，又稱退出狀態、返回碼）會被 Slurm 擷取並儲存在作業記錄中。結束代碼是 0-255 的 8 位元無符號整數，0 表示成功，非零表示失敗。衍生結束代碼（derived exit code）是作業中所有步驟的最高結束代碼，可使用 `sjobexitmod` 指令修改。

---

## 翻譯

### 作業結束代碼

作業的結束代碼（又稱退出狀態、返回碼和完成代碼）會被 Slurm 擷取並儲存為作業記錄的一部分。對於 sbatch 作業，擷取的結束代碼是批次腳本的輸出。對於 salloc 作業，結束代碼將是終止 salloc 工作階段的 exit 呼叫的返回值。對於 srun，結束代碼將是 srun 執行的指令的返回值。

任何非零結束代碼都會被視為作業失敗，並導致作業狀態為 FAILED，原因為「NonZeroExitCode」。

結束代碼是 0 到 255 之間的 8 位元無符號整數。雖然作業可能返回負數結束代碼，但 Slurm 會將其顯示為 0-255 範圍內的無符號值。

### 作業步驟結束代碼

當作業包含多個作業步驟時，每個由 srun 呼叫的可執行檔的結束代碼會分別儲存到作業步驟記錄中。

### 被信號終止的作業

當作業或步驟收到導致其終止的信號時，Slurm 也會擷取信號編號並儲存到作業或步驟記錄中。

### 顯示結束代碼和信號

Slurm 在 `scontrol show job` 和 `sview` 工具程式的輸出中顯示作業的結束代碼。Slurm 在 `scontrol show step` 和 `sview` 工具程式的輸出中顯示作業步驟的結束代碼。

當信號導致作業或步驟終止時，信號編號會顯示在結束代碼之後，以冒號（:）分隔。

### 資料庫作業/步驟記錄

當安裝了 Slurm accounting_storage 外掛程式時，Slurm 控制守護程式會將作業和步驟記錄發送到 Slurm 資料庫。發送到 Slurm 資料庫的作業和步驟記錄可以使用 `sacct` 指令查看。預設的 `sacct` 輸出包含 ExitCode 欄位，其格式與上述 `scontrol` 和 `sview` 的輸出相同。

---

### 衍生結束代碼和註解字串

閱讀上述作業結束代碼的描述後，可以想像這樣一個情境：批次作業的核心任務失敗了，但腳本返回結束代碼零，表示成功。在許多情況下，使用者可能需要檢查作業的輸出檔案後才能確定作業的成功或失敗。

作業包含一個「衍生結束代碼」（derived exit code）欄位。它最初設定為作業所有步驟（srun 呼叫）返回的最高結束代碼值。作業的衍生結束代碼由 Slurm 控制守護程式決定，並在啟用 accounting_storage 外掛程式時發送到資料庫。

除了衍生結束代碼外，Slurm 資料庫中的作業記錄還包含一個註解字串。這會初始化為作業的註解字串（當 slurm.conf 中的 AccountingStoreFlags 參數包含 'job_comment' 時），且只能由使用者修改。

`sacctmgr` 指令新增了一個選項，讓使用者能夠修改作業記錄的這兩個欄位。不允許對作業記錄進行其他修改。對於偏好使用專門設計來查看和修改衍生結束代碼和註解字串的簡單指令的使用者，已建立了 `sjobexitmod` 包裝程式（見下文）。

使用者現在可以在作業完成後為作業的結束代碼加上註解，並提供失敗原因的描述。這包括為看起來失敗但實際上成功的作業註解成功完成的能力。

### sjobexitmod 指令

`sjobexitmod` 指令可用於顯示和更新 Slurm 資料庫作業記錄的兩個衍生結束欄位。`sjobexitmod` 可先用於顯示作業現有的結束代碼/字串：

```bash
> sjobexitmod -l 123
JobID Account NNodes NodeList     State ExitCode DerivedExitCode Comment
----- ------- ------ -------- --------- -------- --------------- -------
123        lc      1     tux0 COMPLETED      0:0             0:0
```

如果需要修改，`sjobexitmod` 可以修改衍生欄位：

```bash
> sjobexitmod -e 49 -r "out of memory" 123

 Modification of job 123 was successful.

> sjobexitmod -l 123
JobID Account NNodes NodeList     State ExitCode DerivedExitCode Comment
----- ------- ------ -------- --------- -------- --------------- -------
123        lc      1     tux0 COMPLETED      0:0            49:0 out of memory
```

現有的 `sacct` 指令也支援兩個新的衍生結束欄位：

```bash
> sacct -X -j 123 -o JobID,NNodes,State,ExitCode,DerivedExitcode,Comment
JobID   NNodes      State ExitCode DerivedExitCode        Comment
------ ------- ---------- -------- --------------- --------------
123          1  COMPLETED      0:0            49:0  out of memory
```

---

## 說明

### 結束代碼的來源

| 作業類型 | 結束代碼來源 |
|----------|--------------|
| sbatch | 批次腳本最後一個指令的結束代碼 |
| salloc | exit 呼叫的返回值 |
| srun | 執行的指令的返回值 |

### 結束代碼格式

Slurm 顯示結束代碼的格式為 `結束代碼:信號編號`：

- `0:0` - 正常成功完成
- `1:0` - 程式以代碼 1 失敗
- `0:9` - 被信號 9 (SIGKILL) 終止
- `0:15` - 被信號 15 (SIGTERM) 終止

### 常見結束代碼含義

| 代碼 | 含義 | 常見原因 |
|------|------|----------|
| 0 | 成功 | 程式正常完成 |
| 1 | 一般錯誤 | 程式內部錯誤 |
| 2 | Shell 誤用 | 指令語法錯誤 |
| 126 | 無法執行 | 權限問題 |
| 127 | 找不到指令 | 路徑錯誤或程式不存在 |
| 128+N | 被信號 N 終止 | 例如 137 = 128+9 (SIGKILL) |
| 130 | Ctrl+C | 被 SIGINT 終止 |
| 137 | SIGKILL | 記憶體超限或被 scancel -s KILL |
| 143 | SIGTERM | 被 scancel 或達到時間限制 |

### 衍生結束代碼 vs 實際結束代碼

| 欄位 | 說明 |
|------|------|
| ExitCode | 主要作業/腳本的實際結束代碼 |
| DerivedExitCode | 所有作業步驟中的最高結束代碼 |

---

## 實務範例

### 查看作業結束代碼

```bash
# 查看特定作業的結束代碼
scontrol show job 12345 | grep -i exit

# 使用 sacct 查看結束代碼
sacct -j 12345 --format=JobID,State,ExitCode,DerivedExitCode

# 查看作業步驟的結束代碼
sacct -j 12345 --format=JobID,JobName,State,ExitCode

# 查看今天所有失敗作業的結束代碼
sacct -S today --state=FAILED -o JobID,JobName,ExitCode,State
```

### 在腳本中正確處理結束代碼

```bash
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --output=%x_%j.out

# 方法 1: 使用 set -e 讓腳本在任何錯誤時退出
set -e

# 執行主要程式
./my_program input.dat

# 方法 2: 手動檢查並返回適當的結束代碼
./my_program input.dat
result=$?

if [ $result -ne 0 ]; then
    echo "Program failed with exit code $result"
    exit $result
fi

# 顯式返回成功
exit 0
```

### 修改衍生結束代碼

```bash
# 查看作業的衍生結束代碼
sjobexitmod -l 12345

# 修改衍生結束代碼並添加註解
sjobexitmod -e 0 -r "Manually verified: output correct" 12345

# 標記實際失敗的作業
sjobexitmod -e 1 -r "Output file corrupted" 12345
```

### 根據結束代碼重新提交

```bash
#!/bin/bash
# 自動重新提交非零結束代碼的作業

JOB_ID=$1
MAX_RETRIES=3

for i in $(seq 1 $MAX_RETRIES); do
    # 提交作業並等待完成
    sbatch --wait my_script.sh
    EXIT_CODE=$?

    if [ $EXIT_CODE -eq 0 ]; then
        echo "Job succeeded on attempt $i"
        exit 0
    fi

    echo "Attempt $i failed with exit code $EXIT_CODE"
    sleep 60  # 等待一分鐘再重試
done

echo "Job failed after $MAX_RETRIES attempts"
exit 1
```

### 分析失敗模式

```bash
# 查看過去一週各種結束代碼的分布
sacct -S $(date -d "7 days ago" +%Y-%m-%d) -u $USER \
    --format=ExitCode -n | sort | uniq -c | sort -rn

# 查看被信號終止的作業
sacct -S today -u $USER --format=JobID,JobName,State,ExitCode | \
    awk -F'[: ]+' '$NF != 0 {print}'
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 腳本總是返回 0 | 確保腳本傳播程式的結束代碼 |
| 忽略步驟失敗 | 使用 `set -e` 或檢查每個重要指令的結束代碼 |
| 不理解 137 代碼 | 137 = 128+9 (SIGKILL)，通常是記憶體超限 |
| 混淆 ExitCode 和 DerivedExitCode | ExitCode 是腳本的，DerivedExitCode 是所有步驟的最高值 |

### 腳本最佳實務

```bash
#!/bin/bash
#SBATCH --job-name=best_practice

# 1. 啟用錯誤檢查
set -e          # 任何指令失敗就退出
set -u          # 使用未定義變數時報錯
set -o pipefail # 管道中任何指令失敗就失敗

# 2. 定義清理函數
cleanup() {
    echo "Cleaning up temporary files..."
    rm -f /tmp/temp_$$_*
}
trap cleanup EXIT

# 3. 主要邏輯
main() {
    echo "Starting job..."

    # 您的程式碼
    ./my_program

    echo "Job completed successfully"
}

# 4. 執行主函數
main "$@"
```

### 除錯技巧

1. **結束代碼 137**：增加 `--mem` 或檢查程式是否有記憶體洩漏
2. **結束代碼 1**：檢查程式輸出和錯誤日誌
3. **結束代碼 127**：確認程式路徑正確、已載入必要的模組
4. **結束代碼 126**：檢查檔案權限 (`chmod +x script.sh`)

---

## 快速參考

### 結束代碼速查表

| 代碼 | 類型 | 說明 | 建議動作 |
|------|------|------|----------|
| 0 | 成功 | 正常完成 | 無 |
| 1-125 | 程式錯誤 | 應用程式定義的錯誤 | 檢查程式文件 |
| 126 | Shell | 權限拒絕 | `chmod +x` |
| 127 | Shell | 找不到指令 | 檢查路徑和模組 |
| 128+N | 信號 | 被信號 N 終止 | 見信號表 |

### 常見信號對應表

| 結束代碼 | 信號 | 名稱 | 常見原因 |
|----------|------|------|----------|
| 130 | 2 | SIGINT | Ctrl+C 或 scancel |
| 137 | 9 | SIGKILL | 記憶體超限、強制終止 |
| 143 | 15 | SIGTERM | 時間限制、正常 scancel |
| 139 | 11 | SIGSEGV | 記憶體區段錯誤 |

### 相關指令

| 指令 | 用途 |
|------|------|
| `scontrol show job` | 查看作業結束代碼 |
| `sacct -o ExitCode` | 查看歷史結束代碼 |
| `sjobexitmod` | 修改衍生結束代碼 |
| `sacctmgr` | 進階作業記錄修改 |
