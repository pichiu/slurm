# Slurm 動態節點 (Dynamic Nodes)

## TL;DR

從 Slurm 22.05 開始，節點可以動態新增和移除。透過 `MaxNodeCount` 設定最大節點數，使用 `slurmd -Z --conf` 動態註冊或 `scontrol create NodeName=...` 建立節點。節點可透過 `scontrol delete nodename=` 刪除。僅支援 `select/cons_tres`，可使用 NodeSet 和 Feature 自動分配分割區。

---

## 翻譯

### 概觀

從 Slurm 22.05 開始，節點可以動態新增和移除。

---

### 動態節點通訊

對於一般非動態建立的節點，Slurm 透過讀取 slurm.conf 來知道如何與節點通訊。這就是為什麼非動態設定中，slurm.conf 在整個叢集中同步非常重要。

對於動態建立的節點，控制器會自動取得動態 slurmd 註冊的節點 **NodeAddr** 和 **NodeHostname**。然後控制器將節點位址傳遞給客戶端，以便它們可以與其他節點通訊，甚至扇出（fanout）。

---

### Slurm 設定

| 參數 | 說明 |
|------|------|
| **MaxNodeCount=#** | 設定系統中可同時活動的最大節點數 |
| **SelectType=select/cons_tres** | 動態節點僅支援 cons_tres |

---

### 分割區指派

動態節點可以在建立時自動指派到分割區，方法是使用：

1. 分割區節點的 **ALL** 關鍵字
2. **NodeSet** 並在節點上指定特性

**範例：**
```
Nodeset=ns1 Feature=f1
Nodeset=ns2 Feature=f2

PartitionName=all  Nodes=ALL Default=yes
PartitionName=dyn1 Nodes=ns1
PartitionName=dyn2 Nodes=ns2
PartitionName=dyn3 Nodes=ns1,ns2
```

---

### 建立節點

節點可以透過兩種方式建立：

#### 方式一：動態 slurmd 註冊

使用 slurmd 的 `-Z` 和 `--conf` 選項，slurmd 會向控制器註冊並自動新增到系統中。

```bash
slurmd -Z --conf "RealMemory=80000 Gres=gpu:2 Feature=f1"
```

#### 方式二：scontrol create

使用 scontrol 建立節點，指定與 slurm.conf 中相同的 **NodeName** 行。

```bash
scontrol create NodeName=d[1-100] CPUs=16 Boards=1 SocketsPerBoard=1 \
  CoresPerSocket=8 ThreadsPerCore=2 RealMemory=31848 Gres=gpu:2 \
  Feature=f1 State=cloud
```

**注意**：
- 僅支援 `State=CLOUD` 和 `State=FUTURE`
- 節點設定應與 slurmd 註冊時的設定相符（例如 slurmd -C）

---

### 刪除節點

使用以下命令刪除節點：

```bash
scontrol delete nodename=<nodelist>
```

**限制**：
- 只有動態節點可以刪除
- 節點上不能有執行中的作業
- 節點不能是保留的一部分

---

### 拓撲

節點可以動態新增到拓撲和從拓撲中移除，詳見[拓撲指南](topology.md)。

---

### 限制

1. 動態節點在內部不會排序，新增到 Slurm 時可能在內部不是按字母順序排列 — 如果節點名稱代表節點的拓撲，這可能導致次優的作業分配。

---

## 說明

### 動態節點 vs 靜態節點

| 特性 | 靜態節點 | 動態節點 |
|------|----------|----------|
| 定義方式 | slurm.conf | slurmd -Z 或 scontrol create |
| 設定同步 | 需要同步 slurm.conf | 控制器自動管理 |
| 通訊方式 | 從 slurm.conf 讀取 | 控制器傳遞位址 |
| 新增/移除 | 需要修改設定檔 | 動態操作 |

### 使用場景

| 場景 | 說明 |
|------|------|
| **雲端爆發** | 按需新增雲端節點處理過量工作負載 |
| **彈性運算** | 根據需求動態調整叢集規模 |
| **臨時資源** | 臨時新增節點處理特定任務 |
| **測試環境** | 快速建立和移除測試節點 |

### 與節能功能整合

動態節點通常與節能（Power Save）功能一起使用：

```
節點狀態流程：
CLOUD（未啟動）→ POWERING_UP → IDLE → 執行作業 → POWERING_DOWN → CLOUD
```

---

## 實務範例

### 基本設定

**slurm.conf：**
```
MaxNodeCount=1000
SelectType=select/cons_tres

# 使用 NodeSet 定義動態節點分組
Nodeset=cloud_gpu Feature=cloud,gpu
Nodeset=cloud_cpu Feature=cloud,cpu

# 分割區設定
PartitionName=all Nodes=ALL Default=yes
PartitionName=gpu Nodes=cloud_gpu
PartitionName=cpu Nodes=cloud_cpu
```

### 動態註冊腳本

**啟動腳本（雲端節點）：**
```bash
#!/bin/bash
# /etc/slurm/start_slurmd.sh

# 偵測硬體設定
CPUS=$(nproc)
MEMORY=$(free -m | awk '/Mem:/ {print $2}')
GPUS=$(nvidia-smi -L | wc -l 2>/dev/null || echo 0)

# 建立設定字串
CONF="CPUs=$CPUS RealMemory=$MEMORY"
if [ "$GPUS" -gt 0 ]; then
    CONF="$CONF Gres=gpu:$GPUS Feature=cloud,gpu"
else
    CONF="$CONF Feature=cloud,cpu"
fi

# 動態註冊
slurmd -Z --conf "$CONF"
```

### 使用 scontrol 批次建立

```bash
# 建立 100 個雲端節點
scontrol create NodeName=cloud[001-100] \
  CPUs=16 RealMemory=64000 \
  Feature=cloud,cpu State=cloud

# 建立 GPU 節點
scontrol create NodeName=gpu[01-20] \
  CPUs=32 RealMemory=128000 Gres=gpu:4 \
  Feature=cloud,gpu State=cloud
```

### 監控動態節點

```bash
# 查看所有動態節點
scontrol show nodes | grep -A 20 "State=CLOUD"

# 查看特定特性的節點
sinfo -N -l -S NodeHost --state=cloud

# 查看節點分配狀態
scontrol show node cloud[001-010]
```

### 自動清理腳本

```bash
#!/bin/bash
# 清理閒置超過 1 小時的雲端節點

IDLE_NODES=$(sinfo -N -h -o "%N %T %E" | \
  awk '$2=="idle" && $3!="none" {print $1}')

for node in $IDLE_NODES; do
    # 檢查閒置時間
    idle_time=$(scontrol show node $node | \
      grep -oP 'Reason=\K[^ ]+' | grep -oP '\d+')

    if [ "$idle_time" -gt 3600 ]; then
        echo "Deleting idle node: $node"
        scontrol delete nodename=$node
    fi
done
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 MaxNodeCount | 必須設定以允許動態節點 |
| 使用非 cons_tres | 動態節點僅支援 select/cons_tres |
| 節點設定不符 | slurmd 註冊設定應與 scontrol create 相符 |
| 刪除有作業的節點 | 等待作業完成或取消作業後再刪除 |
| 節點名稱衝突 | 確保動態節點名稱不與靜態節點重複 |

### 最佳實務

1. **使用 Feature 組織節點**：
   ```
   Feature=cloud,gpu,large_memory
   ```

2. **設定合理的 MaxNodeCount**：
   - 預估最大可能的節點數
   - 留有餘裕但不要過大

3. **監控節點狀態**：
   ```bash
   sinfo -N -l --state=cloud,idle,alloc
   ```

4. **整合節能功能**：
   ```
   SuspendProgram=/etc/slurm/suspend.sh
   ResumeProgram=/etc/slurm/resume.sh
   SuspendTime=300
   ```

### 除錯技巧

```bash
# 測試動態註冊
slurmd -Z --conf "CPUs=4 RealMemory=8000 Feature=test" -D -vvv

# 查看控制器日誌
grep -i "dynamic\|node" /var/log/slurm/slurmctld.log

# 檢查節點通訊
scontrol ping

# 驗證節點設定
slurmd -C
```

---

## 快速參考

### slurm.conf 設定

```
# 必要設定
MaxNodeCount=1000
SelectType=select/cons_tres

# NodeSet 定義
Nodeset=dynamic_nodes Feature=dynamic

# 分割區設定
PartitionName=dynamic Nodes=dynamic_nodes
```

### slurmd 動態註冊

```bash
# 基本註冊
slurmd -Z --conf "RealMemory=80000"

# 含 GPU
slurmd -Z --conf "RealMemory=80000 Gres=gpu:2 Feature=gpu"

# 含多個特性
slurmd -Z --conf "RealMemory=80000 Feature=cloud,large"
```

### scontrol 命令

| 命令 | 功能 |
|------|------|
| `scontrol create NodeName=...` | 建立動態節點 |
| `scontrol delete nodename=...` | 刪除動態節點 |
| `scontrol show node <name>` | 顯示節點資訊 |
| `scontrol update NodeName=... State=...` | 更新節點狀態 |

### 支援的 State 值

| State | 說明 |
|-------|------|
| CLOUD | 雲端節點（未啟動）|
| FUTURE | 未來節點 |

### 相關文件

- [節能指南](power_save.md) - 節點電源管理
- [拓撲](topology.md) - 網路拓撲設定
- [slurm.conf](slurm.conf.html) - 主要設定檔
- [slurmd](slurmd.html) - slurmd 選項
