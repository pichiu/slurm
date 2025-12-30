# Slurm 資源綁定 (Resource Binding)

## TL;DR

Slurm 提供多層級的預設資源綁定設定。優先順序由高到低為：(1) srun `--cpu-bind` 選項、(2) 節點 `CpuBind` 設定、(3) 分割區 `CpuBind` 設定、(4) `TaskPluginParam` 設定。任務可綁定到執行緒、核心、插槽、NUMA 或板卡等資源。

---

## 翻譯

### 概觀

Slurm 有豐富的選項來控制任務到資源的預設綁定。例如，任務可以綁定到個別執行緒、核心、插槽、NUMA 或板卡。請參閱 slurm.conf 和 srun 手冊頁以了解這些選項的運作方式。本文件著重於如何設定預設綁定配置。

預設綁定可以在每個節點、每個分割區或全域層級進行設定。最高優先順序是使用 srun `--cpu-bind` 選項指定的綁定。次高優先順序是節點特定的綁定，前提是作業分配中的任何節點具有某些 `CpuBind` 設定參數，且作業分配中的所有其他節點具有相同或沒有 `CpuBind` 設定參數。再次高優先順序是分割區特定的 `CpuBind` 設定參數（如果有）。最低優先順序是由 `TaskPluginParam` 設定參數指定的綁定。

### 執行優先順序摘要

1. srun `--cpu-bind` 選項
2. 節點 `CpuBind` 設定參數（如果所有節點匹配）
3. 分割區 `CpuBind` 設定參數
4. `TaskPluginParam` 設定參數

---

### Srun --cpu-bind 選項

srun `--cpu-bind` 選項將始終用於控制任務綁定。如果 `--cpu-bind` 選項只包含 "verbose" 而不是識別要綁定的實體，則 verbose 選項將與基於下述 Slurm 設定參數的預設實體一起使用。

---

### 節點 CpuBind 設定

資源綁定資訊的下一個可能來源是節點設定的 `CpuBind` 值，但僅當每個節點具有相同的 `CpuBind` 值（或沒有設定 `CpuBind` 值）時才適用。節點的 `CpuBind` 值在 slurm.conf 檔案中設定。其值可以使用 scontrol 命令查看或修改。

**清除節點的 CpuBind 值：**
```bash
scontrol update NodeName=node01 CpuBind=off
```

---

### 分割區 CpuBind 設定

資源綁定資訊的下一個可能來源是分割區設定的 `CpuBind` 值。分割區的 `CpuBind` 值在 slurm.conf 檔案中設定。其值可以使用 scontrol 命令查看或修改，類似於修改節點的 `CpuBind` 值的方式：

```bash
scontrol update PartitionName=debug CpuBind=cores
```

---

### TaskPluginParam 設定

資源綁定資訊的最後一個可能來源是 slurm.conf 檔案中的 `TaskPluginParam` 設定參數。

---

## 說明

### 綁定層級

| 綁定類型 | 說明 | 適用場景 |
|----------|------|----------|
| `threads` | 綁定到硬體執行緒 | 超執行緒系統 |
| `cores` | 綁定到核心 | 一般多核心系統（推薦） |
| `sockets` | 綁定到插槽 | NUMA 優化 |
| `ldoms` | 綁定到 NUMA 網域 | NUMA 感知應用 |
| `boards` | 綁定到板卡 | 多板卡系統 |
| `none` | 不綁定 | 需要完全彈性的應用 |

### 優先順序視覺化

```
最高優先順序
    │
    ▼
┌─────────────────────────┐
│ srun --cpu-bind 選項    │ ← 使用者層級
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ 節點 CpuBind 設定       │ ← 節點層級
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ 分割區 CpuBind 設定     │ ← 分割區層級
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ TaskPluginParam 設定    │ ← 全域層級
└─────────────────────────┘
    │
    ▼
最低優先順序
```

### CpuBind 參數值

| 值 | 說明 |
|----|------|
| `none` | 不綁定任務到 CPU |
| `boards` | 綁定到板卡 |
| `sockets` | 綁定到插槽 |
| `ldoms` | 綁定到 NUMA 網域 |
| `cores` | 綁定到核心 |
| `threads` | 綁定到執行緒 |
| `off` | 清除/停用綁定設定 |
| `verbose` | 報告綁定資訊 |

---

## 實務範例

### 使用 srun --cpu-bind

```bash
# 綁定到核心
srun --cpu-bind=cores ./my_program

# 綁定到插槽
srun --cpu-bind=sockets ./my_program

# 綁定到執行緒並顯示詳細資訊
srun --cpu-bind=verbose,threads ./my_program

# 使用 CPU 遮罩明確綁定
srun --cpu-bind=mask_cpu:0x0f ./my_program

# 不綁定
srun --cpu-bind=none ./my_program
```

### 設定節點綁定

```bash
# 設定節點的 CPU 綁定為核心
scontrol update NodeName=node01 CpuBind=cores

# 設定多個節點
scontrol update NodeName=node[01-10] CpuBind=sockets

# 清除節點的綁定設定
scontrol update NodeName=node01 CpuBind=off

# 查看節點設定
scontrol show node node01 | grep CpuBind
```

### 設定分割區綁定

```bash
# 設定分割區的 CPU 綁定
scontrol update PartitionName=compute CpuBind=cores

# 設定為插槽綁定
scontrol update PartitionName=gpu CpuBind=sockets

# 查看分割區設定
scontrol show partition compute | grep CpuBind
```

### slurm.conf 設定範例

```
# 節點設定
NodeName=node[01-10] CpuBind=cores Sockets=2 CoresPerSocket=8

# 分割區設定
PartitionName=compute Nodes=node[01-10] CpuBind=cores Default=YES
PartitionName=memory Nodes=node[01-10] CpuBind=sockets

# 全域設定（作為後備）
TaskPluginParam=Cores
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 混合節點有不同的 CpuBind | 節點綁定只在所有節點匹配時生效 |
| 未啟用 TaskPlugin | 確保設定 `TaskPlugin=task/affinity` 或 `task/cgroup` |
| 綁定與應用不匹配 | 根據應用特性選擇適當的綁定層級 |
| 過度綁定造成效能問題 | 某些應用在較寬鬆的綁定下表現更好 |

### 選擇綁定層級的建議

1. **計算密集型應用**：使用 `cores` 綁定
2. **記憶體密集型應用**：使用 `sockets` 或 `ldoms` 綁定
3. **混合應用（MPI+OpenMP）**：使用 `cores` 配合適當的 `--cpus-per-task`
4. **需要最大彈性**：使用 `none`

### 除錯技巧

```bash
# 查看實際綁定
srun --cpu-bind=verbose ./program

# 在作業內查看 CPU 親和性
cat /proc/self/status | grep Cpus_allowed_list

# 查看節點設定
scontrol show node -d | grep CpuBind

# 查看分割區設定
scontrol show partition | grep CpuBind
```

---

## 快速參考

### --cpu-bind 常用值

| 值 | 說明 |
|----|------|
| `none` | 不綁定 |
| `threads` | 綁定到執行緒 |
| `cores` | 綁定到核心 |
| `sockets` | 綁定到插槽 |
| `ldoms` | 綁定到 NUMA 網域 |
| `verbose` | 顯示綁定資訊 |
| `quiet` | 靜默模式（預設） |

### 設定層級優先順序

```
srun --cpu-bind > 節點 CpuBind > 分割區 CpuBind > TaskPluginParam
```

### 相關設定檔

| 設定 | 位置 | 用途 |
|------|------|------|
| `CpuBind` | slurm.conf (NodeName) | 節點層級預設綁定 |
| `CpuBind` | slurm.conf (PartitionName) | 分割區層級預設綁定 |
| `TaskPluginParam` | slurm.conf | 全域預設綁定 |
| `TaskPlugin` | slurm.conf | 啟用綁定功能 |
