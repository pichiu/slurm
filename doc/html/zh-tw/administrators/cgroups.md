# Slurm 控制群組 (Control Group)

---

## TL;DR

控制群組（Control Group，cgroup）是 Linux 核心提供的資源隔離機制。Slurm 使用 cgroup 來限制作業、步驟和任務的資源（CPU、記憶體、裝置），並收集帳務統計。主要外掛包括：`proctrack/cgroup`（程序追蹤）、`task/cgroup`（資源限制）、`jobacct_gather/cgroup`（帳務統計）。Slurm 支援 cgroup v1（已棄用）和 cgroup v2（建議使用）。

---

## Translation（翻譯）

### 目錄

- [控制群組概述](#控制群組概述)
- [Slurm cgroup 外掛設計](#slurm-cgroup-外掛設計)
- [Slurm 中 cgroup 的使用](#slurm-中-cgroup-的使用)
- [Slurm Cgroup 配置概述](#slurm-cgroup-配置概述)
- [目前可用的 Cgroup 外掛](#目前可用的-cgroup-外掛)
  - [proctrack/cgroup 外掛](#proctrackcgroup-外掛)
  - [task/cgroup 外掛](#taskcgroup-外掛)
  - [jobacct_gather/cgroup 外掛](#jobacct_gathercgroup-外掛)
- [使用 cgroup 進行資源專用化](#使用-cgroup-進行資源專用化)
- [Slurm cgroup 外掛](#slurm-cgroup-外掛-1)
  - [cgroup/v1 和 cgroup/v2 的主要差異](#cgroupv1-和-cgroupv2-的主要差異)
  - [控制器介面的主要差異](#控制器介面的主要差異)
  - [其他一般性說明](#其他一般性說明)

### 控制群組概述

控制群組（Control Group）是核心提供的一種機制，用於以階層方式組織程序，並以受控和可配置的方式沿著階層分配系統資源。Slurm 可以使用 cgroups 來限制作業、步驟和任務的不同資源，並獲取這些資源的帳務資訊。

cgroup 為不同的資源提供不同的控制器（以前稱為「子系統」）。Slurm 外掛可以使用其中幾個控制器，例如：*memory、cpu、devices、freezer、cpuset、cpuacct*。每個啟用的控制器都能夠將資源限制到一組程序。如果系統上某個控制器不可用，則 Slurm 無法透過 cgroup 限制相關資源。

「cgroup」代表「control group」，從不大寫。單數形式用於指代整個功能，也用作限定詞，如「cgroup 控制器」。當明確指代多個個別控制群組時，使用複數形式「cgroups」。

Slurm 支援兩種 cgroup 模式：傳統模式（Legacy mode，cgroup v1）和統一模式（Unified Mode，cgroup v2）。不支援混合模式（將版本 1 和版本 2 的控制器混合在系統中）。

**注意**：cgroup/v1 外掛已棄用，在未來的 Slurm 版本中將不再支援。較新的 GNU/Linux 發行版正在放棄或已經放棄對 cgroup v1 的支援，甚至可能不提供所需 cgroup v1 介面的核心支援。Systemd 也棄用了 cgroup v1。從 Slurm 25.05 版本開始，不會為 cgroup v1 新增新功能。將提供對關鍵錯誤的支援，直到最終移除。

### Slurm cgroup 外掛設計

有關 Slurm 內部 Cgroup 外掛的擴展資訊，請閱讀 cgroup/v2 外掛文件。

### Slurm 中 cgroup 的使用

Slurm 提供多個外掛的 cgroup 版本：

| 外掛 | 功能 |
|------|------|
| `proctrack/cgroup` | 程序追蹤和管理 |
| `task/cgroup` | 在步驟和任務層級限制資源 |
| `jobacct_gather/cgroup` | 收集統計資訊 |

cgroups 也可用於資源專用化（將常駐程式限制到核心或記憶體）。

### Slurm Cgroup 配置概述

Slurm cgroups 有幾組配置選項：

| 配置檔 | 用途 |
|--------|------|
| `slurm.conf` | 提供啟用 cgroup 外掛的選項。每個外掛可以獨立於其他外掛啟用或停用 |
| `cgroup.conf` | 提供所有 cgroup 外掛通用的一般選項，以及僅適用於特定外掛的額外選項 |
| 節點配置參數 | 啟用系統級資源專用化 |

### 目前可用的 Cgroup 外掛

#### proctrack/cgroup 外掛

proctrack/cgroup 外掛是 proctrack/linux 等其他 proctrack 外掛的替代方案，用於程序追蹤和暫停/恢復功能。

proctrack/cgroup 使用 freezer 控制器來追蹤作業的所有 pid。它基本上將 pid 儲存在 cgroup 樹中的特定階層中，並在收到指示時負責向這些 pid 發送訊號。例如，如果使用者決定取消作業，Slurm 將透過呼叫 proctrack 外掛並要求它向作業發送 SIGTERM 來在內部執行此命令。由於 proctrack 在 cgroup 中維護所有與 Slurm 相關的 pid 的階層，它將很容易知道需要向哪些 pid 發送訊號。

Proctrack 還可以回應查詢以獲取作業或步驟的所有 pid 列表。

**啟用此外掛：**

```bash
# slurm.conf
ProctrackType=proctrack/cgroup
```

此外掛在 cgroup.conf 中沒有特定選項，但一般選項適用。

#### task/cgroup 外掛

task/cgroup 外掛允許將資源限制到作業、步驟或任務。這是唯一可以確保不違反分配邊界的外掛。

**task/cgroup 提供以下功能：**

- 將作業和步驟限制到其分配的 cpuset
- 將作業和步驟限制到特定的記憶體資源
- 將作業、步驟和任務限制到其分配的 gres，包括 GPU

task/cgroup 外掛使用 cpuset、memory 和 devices 子系統。

**啟用此外掛：**

```bash
# slurm.conf
TaskPlugin=task/cgroup
```

此外掛可以與其他任務外掛堆疊，例如與 task/affinity：

```bash
TaskPlugin=task/cgroup,task/affinity
```

#### jobacct_gather/cgroup 外掛

jobacct_gather/cgroup 外掛是 jobacct_gather/linux 外掛的替代方案，用於收集作業、步驟和任務的帳務統計資訊。

jobacct_gather/cgroup 使用 cpuacct 和 memory cgroup 控制器。

此外掛收集的 cpu 和記憶體統計資訊與 jobacct_gather/linux 收集的 cpu 和記憶體統計資訊不代表相同的資源。cgroup 外掛只是讀取包含整個 pid 子樹資訊的 cgroup.stats 檔案和類似檔案，而 linux 外掛從每個 pid 的 /proc/pid/stat 獲取資訊，然後進行計算，因此效率比 cgroup 稍低（儘管在實踐中不明顯）。

**啟用此外掛：**

```bash
# slurm.conf
JobacctGatherType=jobacct_gather/cgroup
```

### 使用 cgroup 進行資源專用化

資源專用化（Resource Specialization）可用於在每個運算節點上保留一部分核心或特定數量的記憶體，供 Slurm 運算節點常駐程式 slurmd 專用。

- 如果使用 cgroup/v1，保留的資源也將被 slurmstepd 程序使用
- 如果使用 cgroup/v2，slurmstepd 不受此資源專用化的限制。相反，slurmstepd 被限制到分配給作業的資源，因為它被視為作業的一部分，其消耗完全取決於作業的拓撲

系統級資源專用化透過特殊的節點配置參數啟用。

### Slurm cgroup 外掛

cgroup v1 和 v2 外掛在組織其階層和回應不同設計限制方面有非常不同的方式。

#### cgroup/v1 和 cgroup/v2 的主要差異

**1. v2 中的統一模式**

在 cgroup/v1 中，每個控制器有一個單獨的階層，這意味著必須為每個啟用的控制器複製和管理作業結構。例如：

```
# cgroup/v1 - 每個控制器有獨立的階層
/sys/fs/cgroup/memory/slurm/uid_1000/job_1/step_0/
/sys/fs/cgroup/freezer/slurm/uid_1000/job_1/step_0/
```

在 cgroup/v2 中有一個統一階層，控制器在同一層級啟用：

```
# cgroup/v2 - 統一階層
/sys/fs/cgroup/system.slice/slurmstepd.scope/job_1/step_0/
```

**2. v2 中的自上而下限制**

資源自上而下分配，cgroup 只有在從父級分配到資源後才能進一步分配該資源。啟用的控制器列在 `cgroup.controllers` 檔案中，子樹中啟用的控制器列在 `cgroup.subtree_control` 中。

**3. v2 中的非內部程序限制**

在 cgroup/v1 中，階層是自由的，可以在樹中建立任何目錄並將 pid 放入其中。在 cgroup/v2 中，核心限制阻止將 pid 新增到非葉子目錄。

**4. cgroup/v2 上的 Systemd 依賴 - slurmd 和 stepd 的分離**

這不是核心限制而是 systemd 決定，它對決定使用 `Delegate=yes` 的服務施加了重要限制。Systemd（pid 1）決定成為 cgroup 階層 `/sys/fs/cgroup` 的完整擁有者，試圖強加單一寫入者設計。

這使得必須在具有「Delegate=yes」的 unit 下啟動 slurmd 和 slurmstepd 程序。

#### 控制器介面的主要差異

| cgroup/v1 | cgroup/v2 |
|-----------|-----------|
| `memory.limit_in_bytes` | `memory.max` |
| `memory.soft_limit_in_bytes` | `memory.high` |
| `memory.memsw_limit_in_bytes` | `memory.swap.max` |
| `memory.swappiness` | 無 |
| `freezer.state` | `cgroup.freeze` |
| `cpuset.cpus` | `cpuset.cpus.effective` 和 `cpuset.cpus` |
| `cpuset.mems` | `cpuset.mems.effective` 和 `cpuset.mems` |
| `cpuacct.stat` | `cpu.stat` |
| `device.*` | eBPF 程式 |

#### 其他一般性說明

**Swap 帳務（cgroup/v1）：**

在使用 cgroup/v1 時，某些配置可能會排除 swap cgroup 帳務。如果需要此功能，請將以下參數新增到核心命令列：

```bash
cgroup_enable=memory swapaccount=1
```

這通常可以放在 `/etc/default/grub` 的 `GRUB_CMDLINE_LINUX` 變數中。更新檔案後必須執行 `update-grub` 之類的指令。

**JoinControllers（已棄用）：**

在某些 Linux 發行版中，可以使用 systemd 參數 JoinControllers（實際上已棄用）。此參數允許在 cgroup/v1 中將多個控制器掛載在單一階層中。但是，Slurm 無法在此配置下正確運作，因此請確保您的 system.conf 不使用 JoinControllers，並且在使用 cgroup/v1 傳統模式時，所有 cgroup 控制器都在單獨的目錄下。

---

## Explanation（解釋）

### Cgroup 的概念

```
┌─────────────────────────────────────────────────────────────────┐
│                    Linux Cgroup 架構                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cgroup 控制器                                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │  CPU    │ │ Memory  │ │ CPUSet  │ │Freezer  │ │ Devices │  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘  │
│       │           │           │           │           │        │
│       └───────────┼───────────┼───────────┼───────────┘        │
│                   ▼                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Slurm Cgroup 階層                     │   │
│  │                                                         │   │
│  │  /sys/fs/cgroup/                                        │   │
│  │      └── system.slice/                                  │   │
│  │              └── slurmstepd.scope/                      │   │
│  │                      ├── job_1/                         │   │
│  │                      │     ├── step_0/                  │   │
│  │                      │     │     ├── task_0/            │   │
│  │                      │     │     └── task_1/            │   │
│  │                      │     └── step_1/                  │   │
│  │                      └── job_2/                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  每個層級可以設定資源限制：                                       │
│  • memory.max - 記憶體限制                                       │
│  • cpuset.cpus - CPU 核心集合                                    │
│  • cgroup.freeze - 凍結程序                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Slurm Cgroup 外掛的作用

```
作業提交
    │
    ▼
┌─────────────────────────────────────────┐
│           proctrack/cgroup              │
│  • 追蹤作業的所有程序                    │
│  • 發送訊號（SIGTERM、SIGKILL 等）       │
│  • 支援暫停/恢復                         │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│            task/cgroup                  │
│  • 限制 CPU（cpuset）                    │
│  • 限制記憶體                            │
│  • 限制裝置（GPU 等）                    │
│  • 確保不超出分配的資源                  │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│        jobacct_gather/cgroup            │
│  • 收集 CPU 使用統計                     │
│  • 收集記憶體使用統計                    │
│  • 提供帳務資料給 sacct                  │
└─────────────────────────────────────────┘
```

---

## Practical Example（實用範例）

### 範例 1：配置 cgroup 外掛

```bash
# slurm.conf - 啟用 cgroup 外掛
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity
JobacctGatherType=jobacct_gather/cgroup
```

### 範例 2：cgroup.conf 配置

```bash
# /etc/slurm/cgroup.conf

# 啟用記憶體限制
ConstrainRAMSpace=yes
AllowedRAMSpace=100

# 啟用 swap 限制
ConstrainSwapSpace=yes
AllowedSwapSpace=0

# 啟用 CPU 集合限制
ConstrainCores=yes

# 啟用裝置（GPU）限制
ConstrainDevices=yes
```

### 範例 3：檢查 cgroup 配置

```bash
# 檢查系統使用的 cgroup 版本
mount | grep cgroup

# cgroup v1 輸出範例：
# cgroup on /sys/fs/cgroup/memory type cgroup (rw,memory)
# cgroup on /sys/fs/cgroup/cpu type cgroup (rw,cpu)

# cgroup v2 輸出範例：
# cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)
```

### 範例 4：查看作業的 cgroup

```bash
# 找到作業的 cgroup 路徑（cgroup v2）
cat /proc/<slurmstepd_pid>/cgroup

# 查看作業的記憶體限制
cat /sys/fs/cgroup/system.slice/slurmstepd.scope/job_*/memory.max

# 查看作業使用的 CPU
cat /sys/fs/cgroup/system.slice/slurmstepd.scope/job_*/cpuset.cpus.effective
```

### 範例 5：除錯 cgroup 問題

```bash
# 檢查 slurmd 日誌中的 cgroup 訊息
grep -i cgroup /var/log/slurm/slurmd.log

# 檢查控制器是否可用
cat /sys/fs/cgroup/cgroup.controllers

# 查看目前啟用的控制器
cat /sys/fs/cgroup/cgroup.subtree_control
```

### 範例 6：設定資源專用化

```bash
# slurm.conf - 為 slurmd 保留資源
# 保留 2 個核心給 slurmd
CoreSpecCount=2

# 或保留 1GB 記憶體給系統
MemSpecLimit=1024
```

### 範例 7：systemd unit 配置

```bash
# /etc/systemd/system/slurmd.service
[Unit]
Description=Slurm node daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/slurmd -D
Delegate=yes

[Install]
WantedBy=multi-user.target
```

---

## Common Mistakes & Tips（常見錯誤與技巧）

### ❌ 常見錯誤

| 錯誤 | 問題 | 解決方案 |
|------|------|----------|
| 使用混合模式 | Slurm 不支援 cgroup v1 和 v2 混合 | 選擇其中一個版本 |
| 未設定 Delegate=yes | systemd 可能重置 cgroup 設定 | 在 slurmd.service 中加入 `Delegate=yes` |
| 記憶體控制器未啟用 | 無法限制記憶體 | 檢查 `cgroup.controllers` |
| swap 帳務未啟用（v1） | 無法追蹤 swap 使用 | 在核心參數加入 `swapaccount=1` |
| 使用 JoinControllers | Slurm 無法正確運作 | 移除 JoinControllers 配置 |

### ✅ 實用技巧

1. **檢查 cgroup 版本**
   ```bash
   # 簡單方法
   stat -fc %T /sys/fs/cgroup/
   # cgroup2fs = v2
   # tmpfs = v1
   ```

2. **遷移到 cgroup v2**
   ```bash
   # 在 GRUB 中設定
   # /etc/default/grub
   GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"

   # 更新 GRUB
   update-grub
   ```

3. **監控 cgroup 資源使用**
   ```bash
   # 查看記憶體使用
   cat /sys/fs/cgroup/.../memory.current

   # 查看 CPU 統計
   cat /sys/fs/cgroup/.../cpu.stat
   ```

4. **除錯資源限制問題**
   ```bash
   # 檢查作業是否被 OOM 殺死
   dmesg | grep -i "out of memory"

   # 檢查 cgroup 記憶體事件
   cat /sys/fs/cgroup/.../memory.events
   ```

5. **最佳化配置**
   ```bash
   # cgroup.conf - 允許一些額外記憶體以避免 OOM
   AllowedRAMSpace=102
   ```

---

## Quick Reference（快速參考）

### slurm.conf 參數

| 參數 | 值 | 說明 |
|------|-----|------|
| `ProctrackType` | `proctrack/cgroup` | 使用 cgroup 追蹤程序 |
| `TaskPlugin` | `task/cgroup` | 使用 cgroup 限制資源 |
| `JobacctGatherType` | `jobacct_gather/cgroup` | 使用 cgroup 收集統計 |

### cgroup.conf 參數

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `ConstrainRAMSpace` | no | 限制 RAM |
| `AllowedRAMSpace` | 100 | 允許的 RAM 百分比 |
| `ConstrainSwapSpace` | no | 限制 swap |
| `AllowedSwapSpace` | 0 | 允許的 swap 百分比 |
| `ConstrainCores` | no | 限制 CPU 核心 |
| `ConstrainDevices` | no | 限制裝置（GPU） |

### cgroup v1 vs v2 比較

| 功能 | cgroup v1 | cgroup v2 |
|------|-----------|-----------|
| 階層結構 | 每控制器分離 | 統一 |
| 狀態 | 已棄用 | 推薦 |
| Systemd 整合 | 部分 | 完整 |
| 裝置控制 | device 控制器 | eBPF |
| Slurm 支援 | 有（棄用中） | 完整 |

### 常用指令

| 指令 | 功能 |
|------|------|
| `mount \| grep cgroup` | 查看 cgroup 掛載 |
| `cat /sys/fs/cgroup/cgroup.controllers` | 查看可用控制器 |
| `cat /proc/PID/cgroup` | 查看程序的 cgroup |
| `systemctl status slurmd` | 檢查 slurmd 狀態 |
