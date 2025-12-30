# Slurm 通用資源（GRES）排程

---

## TL;DR

通用資源（Generic Resource, GRES）是 Slurm 用於管理 GPU、MPS、頻寬等特殊硬體資源的機制。透過 `slurm.conf` 的 `GresTypes` 和 `Gres` 參數設定，並在 `gres.conf` 中定義詳細配置。作業使用 `--gres=gpu:2` 或 `--gpus=2` 請求資源。支援自動偵測（AutoDetect）、MPS 共享、MIG 分割和 Sharding 等進階功能。

---

## Translation（翻譯）

### 目錄

- [概述](#概述)
- [配置](#配置)
- [執行作業](#執行作業)
- [自動偵測](#自動偵測)
- [Socket 親和性](#socket-親和性)
- [帳務](#帳務)
- [GPU 管理](#gpu-管理)
- [MPS 管理](#mps-管理)
- [MIG 管理](#mig-管理)
- [Sharding](#sharding)

### 概述

Slurm 支援定義和排程任意通用資源（Generic RESources，GRES）的能力。透過可擴展的外掛機制，特定 GRES 類型（包括圖形處理器（Graphics Processing Units，GPU）、CUDA 多程序服務（Multi-Process Service，MPS）裝置和 Sharding）可啟用額外的內建功能。

### 配置

預設情況下，叢集配置中不啟用任何 GRES。您必須在 `slurm.conf` 配置檔中明確指定要管理哪些 GRES。相關的配置參數是 **GresTypes** 和 **Gres**。

請注意，每個節點的 GRES 規格與其他受管理的資源運作方式相同。發現資源少於配置的節點將被置於 DRAIN 狀態。

**slurm.conf 範例片段：**

```bash
# 配置四個 GPU（含 MPS）以及頻寬
GresTypes=gpu,mps,bandwidth
NodeName=tux[0-7] Gres=gpu:tesla:2,gpu:kepler:2,mps:400,bandwidth:lustre:no_consume:4G
```

每個具有通用資源的運算節點通常包含一個 `gres.conf` 檔案，描述該節點上可用的資源、其數量、相關的裝置檔案以及應與這些資源一起使用的核心。

有些情況下，您可能想要在節點上定義通用資源而不指定該 GRES 的數量。例如，節點的檔案系統類型不會隨著作業在該節點上執行而減少。您可以使用 **no_consume** 旗標，允許使用者請求 GRES 而不需要定義在請求時被使用的計數。

### 執行作業

除非在作業提交時使用以下選項明確請求，否則作業不會被分配任何通用資源：

| 選項 | 說明 |
|------|------|
| `--gres` | 每個節點所需的通用資源 |
| `--gpus` | 每個作業所需的 GPU |
| `--gpus-per-node` | 每個節點所需的 GPU，等同於 GPU 的 `--gres` 選項 |
| `--gpus-per-socket` | 每個 socket 所需的 GPU，需要作業指定任務 socket |
| `--gpus-per-task` | 每個任務所需的 GPU，需要作業指定任務數量 |

所有這些選項都被 `salloc`、`sbatch` 和 `srun` 指令支援。

請注意，所有 `--gpu*` 選項僅由 Slurm 的 select/cons_tres 外掛支援。當 select/cons_tres 外掛**未**配置時，請求這些選項的作業將被拒絕。

`--gres` 選項需要一個參數，指定需要哪些通用資源以及使用 `name[:type:count]` 格式需要多少資源，而所有 `--gpu*` 選項需要 `[type]:count` 格式的參數。

- **name**：與 GresTypes 和 Gres 配置參數指定的名稱相同
- **type**：識別該通用資源的特定類型（例如特定型號的 GPU）
- **count**：指定需要多少資源，預設值為 1

例如：`sbatch --gres=gpu:kepler:2 ...`

#### 額外的 GPU 選項

| 選項 | 說明 |
|------|------|
| `--cpus-per-gpu` | 每個 GPU 分配的 CPU 數量 |
| `--gpu-bind` | 定義任務如何綁定到 GPU |
| `--gpu-freq` | 指定 GPU 頻率和/或 GPU 記憶體頻率 |
| `--mem-per-gpu` | 每個 GPU 分配的記憶體 |

作業會根據需要分配特定的通用資源以滿足請求。如果作業被暫停，這些資源不會變為可供其他作業使用。

作業步驟可以使用 `srun` 指令的 `--gres` 選項從分配給作業的資源中分配通用資源。預設情況下，作業步驟將被分配作業請求的所有通用資源。如果需要，作業步驟可以明確指定與作業不同的通用資源數量。

**範例：**

```bash
#!/bin/bash
#
# gres_test.bash
# 提交方式：
# sbatch --gres=gpu:4 -n4 -N1-1 gres_test.bash
#
srun --gres=gpu:2 -n2 --exclusive show_device.sh &
srun --gres=gpu:1 -n1 --exclusive show_device.sh &
srun --gres=gpu:1 -n1 --exclusive show_device.sh &
wait
```

### 自動偵測

如果在 `gres.conf` 中設定 `AutoDetect=nvml`、`AutoDetect=nvidia`、`AutoDetect=rsmi`、`AutoDetect=nrt` 或 `AutoDetect=oneapi`，配置詳細資訊將自動填入任何系統偵測到的 GPU。這消除了在 gres.conf 中明確配置 GPU 的需要，但 slurm.conf 中的 `Gres=` 行仍然需要，以便告訴 slurmctld 預期有多少 GRES。

**注意事項：**
- `AutoDetect=nvml`、`AutoDetect=rsmi` 和 `AutoDetect=oneapi` 需要在節點上安裝對應的 GPU 管理程式庫，並在 Slurm 配置期間找到
- `AutoDetect=nvml` 和 `AutoDetect=nvidia` 都可以偵測 NVIDIA GPU
- `AutoDetect=nvidia`（在 Slurm 24.11 中新增）不需要安裝 nvml 程式庫，但不偵測 MIG 或 NVlink

**gres.conf 範例：**

```bash
# 配置四個 GPU（含 MPS）以及頻寬
AutoDetect=nvml
Name=gpu Type=gp100  File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gp100  File=/dev/nvidia1 Cores=0,1
Name=gpu Type=p6000  File=/dev/nvidia2 Cores=2,3
Name=gpu Type=p6000  File=/dev/nvidia3 Cores=2,3
Name=mps Count=200  File=/dev/nvidia0
Name=mps Count=200  File=/dev/nvidia1
Name=mps Count=100  File=/dev/nvidia2
Name=mps Count=100  File=/dev/nvidia3
Name=bandwidth Type=lustre Count=4G Flags=CountOnly
```

要查看偵測到的 GPU 及其名稱，執行：
```bash
$ slurmd -C
NodeName=node0 ... Gres=gpu:geforce_rtx_2060:1 ...
Found gpu:geforce_rtx_2060:1 with Autodetect=nvml (可以使用 GPU 名稱的子字串)
UpTime=...
```

要測試配置，可執行：`slurmd -G`

### Socket 親和性配置

Slurm 的作業排程器在內部以 socket 為基礎處理 GRES 親和性。但是，`gres.conf` 介面允許管理員為 GPU 親和性配置指定 `Cores`。這產生了一個常見的誤解，認為 Slurm 會在作業排程期間遵守核心級別的親和性，但實際上它從未這樣做。

當使用 `AutoDetect=nvml` 或其他 GPU 自動偵測方法時，GPU 核心親和性會根據 GPU 驅動程式報告的硬體拓撲自動偵測。在許多具有複雜快取階層的現代系統上，自動偵測的 GPU 親和性可能與 Slurm 的 socket 邊界不一致。

**建議的解決方案**是配置 Slurm 使其 socket 定義與硬體拓撲一致，使用拓撲參數如 `l3cache_as_socket` 或 `numa_node_as_socket`。

```bash
# 測試硬體偵測
$ slurmd -C --parameters=l3cache_as_socket
NodeName=node0 CPUs=32 Boards=1 SocketsPerBoard=8 CoresPerSocket=2 ThreadsPerCore=2 ...

# 全域配置
SlurmdParameters=l3cache_as_socket

# 或每節點配置
NodeName=node0 CPUs=32 Boards=1 SocketsPerBoard=8 CoresPerSocket=2 ThreadsPerCore=2 Parameters=l3cache_as_socket Gres=gpu:a100:2
```

### 帳務

GPU 記憶體和 GPU 使用率可以作為 TRES 追蹤使用 GPU 資源的任務。如果配置了 `AccountingStorageTRES=gres/gpu`，則 gres/gpumem 和 gres/gpuutil 將自動配置並從 GPU 作業收集。

gres/gpumem 和 gres/gpuutil 僅適用於使用 `AutoDetect=nvml` 的 NVIDIA GPU 和使用 `AutoDetect=rsmi` 的 AMD GPU。

**slurm.conf 範例：**
```bash
AccountingStorageTres=gres/gpu
NodeName=n1 Gres=gpu:a100:2
```

**gres.conf 範例：**
```bash
AutoDetect=nvml
```

**查看使用率範例：**
```bash
$ sacct -j 1277.0 --format=tresusageinave -p
TRESUsageInAve|
cpu=00:00:11,energy=0,fs/disk=87613,gres/gpumem=36266M,gres/gpuutil=100,mem=628748K,pages=0,vmem=0|
```

### GPU 管理

對於 Slurm 的 GPU GRES 外掛，環境變數 `CUDA_VISIBLE_DEVICES` 會為每個作業步驟設定，以決定每個節點上哪些 GPU 可供使用。

CUDA 3.1 版本（或更高版本）使用此環境變數，以便在具有 GPU 的節點上執行多個作業或作業步驟，並確保分配給每個的資源是唯一的。

**範例輸出：**
```
JobStep=1234.0 CUDA_VISIBLE_DEVICES=0,1
JobStep=1234.1 CUDA_VISIBLE_DEVICES=2
JobStep=1234.2 CUDA_VISIBLE_DEVICES=3
```

**注意事項：**
- 確保在 `gres.conf` 中指定 `File` 參數，並確保它們按遞增數字順序排列
- `CUDA_VISIBLE_DEVICES` 環境變數也會在作業的 Prolog 和 Epilog 程式中設定
- 為使 NVML 報告的編號與 CUDA 報告的編號匹配，需設定 `CUDA_DEVICE_ORDER=PCI_BUS_ID`

### MPS 管理

CUDA 多程序服務（Multi-Process Service，MPS）提供了一種機制，讓多個作業可以共享 GPU，每個作業被分配 GPU 資源的某個百分比。

節點上可用的 MPS 資源總數應在 `slurm.conf` 檔案中配置：
```bash
NodeName=tux[1-16] Gres=gpu:2,mps:200
```

**MPS 配置選項：**

1. **無 MPS 配置**：slurm.conf 中定義的 gres/mps 數量將平均分配到節點上配置的所有 GPU
2. **僅 Name 和 Count 參數**：gres/mps 數量將平均分配到所有 GPU
3. **Name、File 和 Count 參數**：每個 File 參數應識別 GPU 的裝置檔案路徑，Count 應識別該特定 GPU 裝置可用的 gres/mps 資源數量

**重要限制：**
- 同一個 GPU 可以作為 GPU 類型的 GRES 或 MPS 類型的 GRES 分配，但不能同時兩者
- 作業不能同時請求 GPU 和 MPS 類型的 GRES
- 請求 MPS 資源的作業不能指定 GPU 頻率

**gres.conf 範例：**
```bash
# 範例：配置四個不同類型的 GPU（含 MPS）
AutoDetect=nvml
Name=gpu Type=gtx1080 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gtx1070 File=/dev/nvidia1 Cores=0,1
Name=gpu Type=gtx1060 File=/dev/nvidia2 Cores=2,3
Name=gpu Type=gtx1050 File=/dev/nvidia3 Cores=2,3
Name=mps Count=1300   File=/dev/nvidia0
Name=mps Count=1200   File=/dev/nvidia1
Name=mps Count=1100   File=/dev/nvidia2
Name=mps Count=1000   File=/dev/nvidia3
```

### MIG 管理

從 21.08 版本開始，Slurm 現在支援 NVIDIA 多實例 GPU（Multi-Instance GPU，MIG）裝置。此功能允許某些較新的 NVIDIA GPU（如 A100）將一個 GPU 分割成最多七個獨立、隔離的 GPU 實例。Slurm 可以將這些 MIG 實例視為單獨的 GPU，具有完整的 cgroup 隔離和任務綁定。

**slurm.conf 範例：**
```bash
AccountingStorageTRES=gres/gpu,gres/gpu:a100,gres/gpu:a100_3g.20gb
GresTypes=gpu
NodeName=tux[1-16] gres=gpu:a100:1,gpu:a100_3g.20gb:2
```

**gres.conf：**
```bash
AutoDetect=nvml
```

**注意**：Slurm 預期 MIG 裝置已經分割完成，不支援動態 MIG 分割。

### Sharding

Sharding 提供了一種通用機制，讓多個作業可以共享 GPU。雖然它允許多個作業在給定的 GPU 上執行，但它不會隔離在 GPU 上執行的程序，僅允許 GPU 被共享。因此，Sharding 最適合同質工作負載。

建議將節點上的 shard 數量限制為等於可以在節點上同時執行的最大作業數（即核心數）。

**slurm.conf 範例：**
```bash
AccountingStorageTRES=gres/gpu,gres/shard
GresTypes=gpu,shard
NodeName=tux[1-16] Gres=gpu:2,shard:64
```

**gres.conf 範例：**
```bash
AutoDetect=nvml
Name=gpu Type=gp100 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gp100 File=/dev/nvidia1 Cores=0,1
# 在 4 個可用 GPU 的每個上設定 gres/shard Count 值為 8
Name=shard Count=32
```

請求 shard 資源的作業將設定 `CUDA_VISIBLE_DEVICES`、`ROCR_VISIBLE_DEVICES` 或 `GPU_DEVICE_ORDINAL` 環境變數。具有 shard 的步驟會設定 `SLURM_SHARDS_ON_NODE` 指示分配的 shard 數量。

---

## Explanation（解釋）

### GRES 的概念

```
┌─────────────────────────────────────────────────────────────────┐
│                      Slurm GRES 架構                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  slurm.conf                      gres.conf                      │
│  ┌─────────────────┐            ┌─────────────────────┐        │
│  │ GresTypes=gpu   │            │ AutoDetect=nvml     │        │
│  │ NodeName=node1  │    ──►     │ Name=gpu Type=a100  │        │
│  │ Gres=gpu:a100:4 │            │ File=/dev/nvidia0   │        │
│  └─────────────────┘            │ Cores=0-15          │        │
│                                 └─────────────────────┘        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    節點 (Node)                           │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │  GPU 0  │ │  GPU 1  │ │  GPU 2  │ │  GPU 3  │       │   │
│  │  │ (a100)  │ │ (a100)  │ │ (a100)  │ │ (a100)  │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  │                                                         │   │
│  │  作業請求: --gres=gpu:2                                  │   │
│  │  結果: CUDA_VISIBLE_DEVICES=0,1                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### GPU 共享模式比較

| 模式 | 隔離程度 | 使用場景 | 限制 |
|------|----------|----------|------|
| **獨佔 GPU** | 完全隔離 | 需要完整 GPU 資源 | 一個作業一個 GPU |
| **MPS** | 部分隔離 | 小型 GPU 作業共享 | 同一使用者限制 |
| **MIG** | 硬體隔離 | 需要保證資源 | 僅支援特定 GPU |
| **Sharding** | 無隔離 | 同質工作負載 | 無資源保證 |

### 環境變數的作用

```
作業提交：sbatch --gres=gpu:2 job.sh
         │
         ▼
┌─────────────────────────────────────────┐
│ Slurm 排程器分配 GPU 0 和 GPU 1          │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ 設定環境變數：                           │
│ CUDA_VISIBLE_DEVICES=0,1                │
│ CUDA_DEVICE_ORDER=PCI_BUS_ID            │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│ CUDA 程式只能看到這兩個 GPU              │
│ 程式中的 GPU 0 = 實際的 GPU 0           │
│ 程式中的 GPU 1 = 實際的 GPU 1           │
└─────────────────────────────────────────┘
```

---

## Practical Example（實用範例）

### 範例 1：基本 GPU 作業

```bash
#!/bin/bash
#SBATCH --job-name=gpu_test
#SBATCH --gres=gpu:1              # 請求 1 個 GPU
#SBATCH --ntasks=1
#SBATCH --time=01:00:00
#SBATCH --output=gpu_%j.out

# 顯示分配的 GPU
echo "分配的 GPU: $CUDA_VISIBLE_DEVICES"

# 使用 nvidia-smi 確認
nvidia-smi

# 執行 GPU 程式
python train_model.py
```

### 範例 2：請求特定類型的 GPU

```bash
#!/bin/bash
#SBATCH --job-name=a100_job
#SBATCH --gres=gpu:a100:2         # 請求 2 個 A100 GPU
#SBATCH --cpus-per-gpu=8          # 每個 GPU 分配 8 個 CPU
#SBATCH --mem-per-gpu=32G         # 每個 GPU 分配 32GB 記憶體
#SBATCH --time=04:00:00

echo "使用 ${SLURM_GPUS_ON_NODE} 個 GPU"
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"

# 執行多 GPU 訓練
python -m torch.distributed.launch --nproc_per_node=2 train.py
```

### 範例 3：多作業步驟共享 GPU

```bash
#!/bin/bash
#SBATCH --job-name=multi_step
#SBATCH --gres=gpu:4              # 總共請求 4 個 GPU
#SBATCH --ntasks=4
#SBATCH --nodes=1

# 作業步驟 1：使用 2 個 GPU
srun --gres=gpu:2 -n2 --exclusive ./task1.sh &

# 作業步驟 2：使用 1 個 GPU
srun --gres=gpu:1 -n1 --exclusive ./task2.sh &

# 作業步驟 3：使用 1 個 GPU
srun --gres=gpu:1 -n1 --exclusive ./task3.sh &

# 等待所有步驟完成
wait
```

### 範例 4：使用 MPS 共享 GPU

```bash
# 請求 MPS 資源（假設配置了 100% = 100 單位）
sbatch --gres=mps:50 job.sh   # 請求 50% 的 GPU

# 多個作業可以共享同一個 GPU
sbatch --gres=mps:30 job1.sh  # 30%
sbatch --gres=mps:30 job2.sh  # 30%
sbatch --gres=mps:40 job3.sh  # 40%（等待前兩個完成後執行）
```

### 範例 5：查看 GPU 配置和狀態

```bash
# 查看節點的 GPU 配置
scontrol show node node001 | grep -i gres

# 查看所有節點的 GPU 使用情況
sinfo -o "%N %G" -N

# 查看特定分割區的 GPU 資訊
sinfo -p gpu -o "%n %G %C %t"

# 查看作業的 GPU 分配
scontrol show job 12345 | grep -i gres

# 使用 sacct 查看 GPU 使用統計
sacct -j 12345 --format=JobID,AllocGRES,ReqGRES,TRESUsageInAve
```

### 範例 6：設定 GPU 頻率

```bash
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH --gpu-freq=high           # 設定高頻率

# 或指定具體頻率
# --gpu-freq=memory=5001,graphics=1530

nvidia-smi -q -d CLOCK
./compute_intensive_task
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| 未配置 select/cons_tres | `--gpus` 選項無法使用 | 在 slurm.conf 設定 `SelectType=select/cons_tres` |
| gres.conf 未配置 File | GPU 無法正確隔離 | 為每個 GPU 指定 `File=/dev/nvidiaX` |
| AutoDetect 核心親和性不匹配 | 節點進入 DRAIN 狀態 | 使用 `l3cache_as_socket` 參數 |
| 同時請求 GPU 和 MPS | 作業被拒絕 | 選擇其中一種資源類型 |
| MPS Count 非 100 的倍數 | 百分比計算不精確 | 使用 100 的倍數作為 MPS Count |

### ✅ 實用技巧

1. **驗證 GPU 配置**
   ```bash
   # 在節點上測試配置
   slurmd -C    # 顯示偵測到的資源
   slurmd -G    # 測試 GRES 配置
   ```

2. **除錯 GPU 分配問題**
   ```bash
   # 查看 slurmd 日誌
   grep -i gres /var/log/slurm/slurmd.log

   # 檢查環境變數
   srun --gres=gpu:1 env | grep -i cuda
   ```

3. **監控 GPU 使用率**
   ```bash
   # 即時監控（在節點上）
   watch -n 1 nvidia-smi

   # 透過 Slurm 帳務
   sreport cluster utilization tres=gres/gpu
   ```

4. **最佳化 GPU 綁定**
   ```bash
   # 將 GPU 綁定到最近的 CPU
   srun --gres=gpu:1 --gpu-bind=closest ./app

   # 或指定綁定方式
   srun --gres=gpu:2 --gpu-bind=map_gpu:0,1 ./app
   ```

5. **使用 cgroup 隔離 GPU**
   ```bash
   # 在 slurm.conf 中配置
   TaskPlugin=task/cgroup
   # 在 cgroup.conf 中
   ConstrainDevices=yes
   ```

---

## Quick Reference（快速參考）

### 作業提交選項

| 選項 | 說明 | 範例 |
|------|------|------|
| `--gres=gpu:N` | 每節點 N 個 GPU | `--gres=gpu:2` |
| `--gres=gpu:type:N` | 每節點 N 個特定類型 GPU | `--gres=gpu:a100:2` |
| `--gpus=N` | 每作業 N 個 GPU | `--gpus=4` |
| `--gpus-per-node=N` | 每節點 N 個 GPU | `--gpus-per-node=2` |
| `--gpus-per-task=N` | 每任務 N 個 GPU | `--gpus-per-task=1` |
| `--cpus-per-gpu=N` | 每 GPU 分配 N 個 CPU | `--cpus-per-gpu=8` |
| `--mem-per-gpu=SIZE` | 每 GPU 分配記憶體 | `--mem-per-gpu=32G` |
| `--gpu-bind=TYPE` | GPU 綁定類型 | `--gpu-bind=closest` |
| `--gpu-freq=FREQ` | 設定 GPU 頻率 | `--gpu-freq=high` |

### 環境變數

| 變數 | 說明 |
|------|------|
| `CUDA_VISIBLE_DEVICES` | NVIDIA GPU 可見裝置 |
| `CUDA_DEVICE_ORDER` | GPU 排序方式 |
| `ROCR_VISIBLE_DEVICES` | AMD GPU 可見裝置 |
| `GPU_DEVICE_ORDINAL` | Intel GPU 可見裝置 |
| `SLURM_GPUS_ON_NODE` | 節點上分配的 GPU 數量 |
| `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` | MPS 執行緒百分比 |
| `SLURM_SHARDS_ON_NODE` | 節點上分配的 shard 數量 |

### 配置參數

| 參數 | 檔案 | 說明 |
|------|------|------|
| `GresTypes` | slurm.conf | 定義 GRES 類型 |
| `Gres` | slurm.conf | 節點的 GRES 配置 |
| `AutoDetect` | gres.conf | 自動偵測模式 |
| `Name` | gres.conf | GRES 名稱 |
| `Type` | gres.conf | GRES 類型/型號 |
| `File` | gres.conf | 裝置檔案路徑 |
| `Cores` | gres.conf | 關聯的 CPU 核心 |
| `Count` | gres.conf | 資源數量 |

### 管理指令

| 指令 | 功能 |
|------|------|
| `sinfo -o "%N %G"` | 顯示節點 GRES |
| `scontrol show node` | 顯示節點詳情（含 GRES） |
| `scontrol show job` | 顯示作業 GRES 分配 |
| `slurmd -C` | 偵測節點 GRES |
| `slurmd -G` | 測試 GRES 配置 |
| `sacct --format=AllocGRES` | 查看作業 GRES 使用 |
