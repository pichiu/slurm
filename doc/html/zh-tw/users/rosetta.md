# 工作負載管理器對照表（Rosetta Stone）

## TL;DR
- 本文件提供 PBS/Torque、Slurm、LSF、SGE 和 LoadLeveler 之間的指令對照
- 從其他排程系統遷移到 Slurm 時的快速參考
- 涵蓋作業提交、管理、查詢和環境變數的對應關係

---

## Translation

本表列出主要工作負載管理系統使用的最常見指令、環境變數和作業規格選項：PBS/Torque、Slurm、LSF、SGE 和 LoadLeveler。每個工作負載管理器都有獨特的功能，但最常用的功能在所有這些環境中都可用。

---

## 使用者指令對照

### 作業提交

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 提交批次作業 | `qsub script.sh` | `sbatch script.sh` | `bsub < script.sh` | `qsub script.sh` |
| 互動式作業 | `qsub -I` | `salloc` / `srun --pty bash` | `bsub -Is bash` | `qlogin` / `qrsh` |
| 啟動平行作業 | `mpirun` / `pbsdsh` | `srun` | `mpirun` / `blaunch` | `mpirun` |
| 作業陣列 | `qsub -t 1-10` | `sbatch --array=1-10` | `bsub -J "name[1-10]"` | `qsub -t 1-10` |

### 作業管理

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 取消作業 | `qdel <jobid>` | `scancel <jobid>` | `bkill <jobid>` | `qdel <jobid>` |
| 保持作業 | `qhold <jobid>` | `scontrol hold <jobid>` | `bstop <jobid>` | `qhold <jobid>` |
| 釋放作業 | `qrls <jobid>` | `scontrol release <jobid>` | `bresume <jobid>` | `qrls <jobid>` |
| 重新排隊 | `qrerun <jobid>` | `scontrol requeue <jobid>` | `brequeue <jobid>` | N/A |
| 修改作業 | `qalter` | `scontrol update` | `bmod` | `qalter` |

### 作業查詢

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 顯示所有作業 | `qstat` | `squeue` | `bjobs` | `qstat` |
| 顯示使用者作業 | `qstat -u <user>` | `squeue -u <user>` | `bjobs -u <user>` | `qstat -u <user>` |
| 顯示作業詳情 | `qstat -f <jobid>` | `scontrol show job <jobid>` | `bjobs -l <jobid>` | `qstat -j <jobid>` |
| 顯示佇列/分割區 | `qstat -Q` | `sinfo` | `bqueues` | `qstat -g c` |
| 顯示節點狀態 | `pbsnodes -a` | `sinfo -N` / `scontrol show nodes` | `bhosts` | `qhost` |

### 會計與歷史

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 作業歷史 | `qstat -x` | `sacct` | `bhist` | `qacct` |
| 顯示公平份額 | N/A | `sshare` | `bshare` | N/A |

---

## 作業規格選項對照

### 資源請求

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 節點數量 | `-l nodes=N` | `-N <N>` / `--nodes=<N>` | `-n <N>` | `-pe mpi <N>` |
| CPU/核心數 | `-l ppn=N` | `-n <N>` / `--ntasks=<N>` | `-n <N>` | `-pe mpi <N>` |
| 每節點 CPU | `-l ppn=N` | `--ntasks-per-node=<N>` | `-R "span[ptile=N]"` | N/A |
| 每任務 CPU | N/A | `--cpus-per-task=<N>` | N/A | N/A |
| 記憶體 | `-l mem=<size>` | `--mem=<size>` | `-M <size>` | `-l h_vmem=<size>` |
| 每 CPU 記憶體 | `-l pmem=<size>` | `--mem-per-cpu=<size>` | `-R "rusage[mem=N]"` | N/A |
| GPU | `-l gpus=N` | `--gres=gpu:<N>` / `--gpus=<N>` | `-R "select[ngpus>0]"` | `-l gpu=<N>` |
| 特定節點 | `-l nodes=<name>` | `-w <nodelist>` | `-m <hostlist>` | `-l h=<hostname>` |
| 排除節點 | N/A | `-x <nodelist>` / `--exclude=<nodelist>` | N/A | N/A |

### 時間與執行

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 時間限制 | `-l walltime=HH:MM:SS` | `-t HH:MM:SS` / `--time=` | `-W HH:MM` | `-l h_rt=HH:MM:SS` |
| 開始時間 | `-a <datetime>` | `--begin=<datetime>` | `-b <datetime>` | `-a <datetime>` |
| 作業依賴 | `-W depend=afterok:<jobid>` | `--dependency=afterok:<jobid>` | `-w "done(<jobid>)"` | `-hold_jid <jobid>` |
| 重新執行 | `-r y` | `--requeue` | `-r` | `-r y` |
| 獨佔節點 | `-l nodes=N:ppn=X -W x=naccesspolicy:SINGLEJOB` | `--exclusive` | `-x` | `-l excl=true` |

### 輸出與環境

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 作業名稱 | `-N <name>` | `-J <name>` / `--job-name=<name>` | `-J <name>` | `-N <name>` |
| 標準輸出 | `-o <file>` | `-o <file>` / `--output=<file>` | `-o <file>` | `-o <file>` |
| 標準錯誤 | `-e <file>` | `-e <file>` / `--error=<file>` | `-e <file>` | `-e <file>` |
| 合併輸出錯誤 | `-j oe` | 預設合併（或用不同 -o -e） | `-o <file>` | `-j y` |
| 分割區/佇列 | `-q <queue>` | `-p <partition>` | `-q <queue>` | `-q <queue>` |
| 帳號 | `-A <account>` | `-A <account>` / `--account=<account>` | `-P <project>` | `-A <account>` |
| QOS | N/A | `--qos=<qos>` | N/A | N/A |
| 郵件通知 | `-m abe` | `--mail-type=ALL` | `-B -N` | `-m beas` |
| 郵件地址 | `-M <email>` | `--mail-user=<email>` | `-u <email>` | `-M <email>` |
| 工作目錄 | `-d <dir>` | `-D <dir>` / `--chdir=<dir>` | (預設為提交目錄) | `-wd <dir>` |

---

## 環境變數對照

| 功能 | PBS/Torque | Slurm | LSF | SGE |
|------|------------|-------|-----|-----|
| 作業 ID | `$PBS_JOBID` | `$SLURM_JOB_ID` / `$SLURM_JOBID` | `$LSB_JOBID` | `$JOB_ID` |
| 作業名稱 | `$PBS_JOBNAME` | `$SLURM_JOB_NAME` | `$LSB_JOBNAME` | `$JOB_NAME` |
| 節點列表 | `$PBS_NODEFILE` | `$SLURM_JOB_NODELIST` | `$LSB_HOSTS` | `$PE_HOSTFILE` |
| 節點數量 | `$PBS_NUM_NODES` | `$SLURM_JOB_NUM_NODES` / `$SLURM_NNODES` | `$LSB_DJOB_NUMPROC` | `$NHOSTS` |
| 任務數量 | `$PBS_NP` / `$PBS_TASKNUM` | `$SLURM_NTASKS` / `$SLURM_NPROCS` | `$LSB_DJOB_NUMPROC` | `$NSLOTS` |
| 任務 ID | `$PBS_VNODENUM` | `$SLURM_PROCID` | N/A | `$SGE_TASK_ID` |
| 本地任務 ID | N/A | `$SLURM_LOCALID` | `$LSB_JOBINDEX` | N/A |
| 提交目錄 | `$PBS_O_WORKDIR` | `$SLURM_SUBMIT_DIR` | `$LS_SUBCWD` | `$SGE_O_WORKDIR` |
| 佇列/分割區 | `$PBS_QUEUE` | `$SLURM_JOB_PARTITION` | `$LSB_QUEUE` | `$QUEUE` |
| 陣列索引 | `$PBS_ARRAYID` | `$SLURM_ARRAY_TASK_ID` | `$LSB_JOBINDEX` | `$SGE_TASK_ID` |
| 陣列作業 ID | N/A | `$SLURM_ARRAY_JOB_ID` | N/A | N/A |
| 每節點 CPU | `$PBS_NUM_PPN` | `$SLURM_CPUS_ON_NODE` | N/A | N/A |
| 每任務 CPU | N/A | `$SLURM_CPUS_PER_TASK` | N/A | N/A |
| 節點名稱 | `$PBS_O_HOST` | `$SLURMD_NODENAME` / `$SLURM_NODELIST` | `$LSB_HOSTS` | `$HOSTNAME` |

---

## Practical Example

### 從 PBS/Torque 遷移到 Slurm

**PBS 腳本：**
```bash
#!/bin/bash
#PBS -N my_job
#PBS -l nodes=4:ppn=8
#PBS -l walltime=02:00:00
#PBS -l mem=16gb
#PBS -q batch
#PBS -o output.log
#PBS -e error.log
#PBS -M user@example.com
#PBS -m abe

cd $PBS_O_WORKDIR
mpirun ./my_program
```

**對應的 Slurm 腳本：**
```bash
#!/bin/bash
#SBATCH -J my_job                    # 作業名稱
#SBATCH -N 4                         # 節點數量
#SBATCH --ntasks-per-node=8          # 每節點任務數
#SBATCH -t 02:00:00                  # 時間限制
#SBATCH --mem=16G                    # 記憶體
#SBATCH -p batch                     # 分割區
#SBATCH -o output.log                # 標準輸出
#SBATCH -e error.log                 # 標準錯誤
#SBATCH --mail-user=user@example.com # 郵件地址
#SBATCH --mail-type=ALL              # 郵件通知

cd $SLURM_SUBMIT_DIR
srun ./my_program
```

### 常用指令轉換範例

```bash
# PBS: 查看所有作業
qstat
# Slurm 對應:
squeue

# PBS: 查看特定使用者作業
qstat -u username
# Slurm 對應:
squeue -u username

# PBS: 查看作業詳情
qstat -f 12345
# Slurm 對應:
scontrol show job 12345

# PBS: 取消作業
qdel 12345
# Slurm 對應:
scancel 12345

# PBS: 保持作業
qhold 12345
# Slurm 對應:
scontrol hold 12345

# PBS: 互動式作業
qsub -I -l nodes=1:ppn=4
# Slurm 對應:
salloc -N1 --ntasks-per-node=4
# 或
srun --pty -N1 -n4 bash
```

---

## Common Mistakes & Tips

### 常見遷移錯誤

1. **混淆 -n 和 -N 選項**
   ```bash
   # PBS: -l nodes=4 表示 4 個節點
   # Slurm: -N4 表示 4 個節點，-n4 表示 4 個任務

   # 正確對應：
   # PBS -l nodes=4:ppn=8 → Slurm -N4 --ntasks-per-node=8
   ```

2. **忘記更換環境變數**
   ```bash
   # 錯誤：在 Slurm 腳本中使用 PBS 變數
   cd $PBS_O_WORKDIR  # 不會工作

   # 正確：使用 Slurm 變數
   cd $SLURM_SUBMIT_DIR
   ```

3. **mpirun vs srun**
   ```bash
   # 在 Slurm 中推薦使用 srun 而非 mpirun
   # PBS: mpirun -np 32 ./program
   # Slurm: srun -n 32 ./program  （推薦）
   ```

### 實用建議

- **保留舊腳本**：遷移時保留原始 PBS 腳本作為參考
- **測試小作業**：先用小規模作業測試轉換後的腳本
- **查閱 man 頁面**：Slurm 有詳細的 man 頁面（`man sbatch`、`man srun`）
- **使用 --test-only**：用 `sbatch --test-only` 驗證腳本而不實際提交

---

## Quick Reference

### PBS → Slurm 快速對照

| PBS | Slurm | 說明 |
|-----|-------|------|
| `qsub` | `sbatch` | 提交批次作業 |
| `qsub -I` | `salloc` / `srun --pty bash` | 互動式作業 |
| `qstat` | `squeue` | 查看作業 |
| `qstat -f` | `scontrol show job` | 作業詳情 |
| `qdel` | `scancel` | 取消作業 |
| `qhold` | `scontrol hold` | 保持作業 |
| `qrls` | `scontrol release` | 釋放作業 |
| `pbsnodes` | `sinfo -N` | 節點狀態 |
| `-l nodes=N` | `-N <N>` | 節點數 |
| `-l ppn=N` | `--ntasks-per-node=<N>` | 每節點任務 |
| `-l walltime=` | `-t` / `--time=` | 時間限制 |
| `-l mem=` | `--mem=` | 記憶體 |
| `-q <queue>` | `-p <partition>` | 佇列/分割區 |
| `-N <name>` | `-J <name>` | 作業名稱 |
| `$PBS_JOBID` | `$SLURM_JOB_ID` | 作業 ID |
| `$PBS_O_WORKDIR` | `$SLURM_SUBMIT_DIR` | 提交目錄 |
