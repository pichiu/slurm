# Slurm 拓撲指南 (Topology Guide)

## TL;DR

Slurm 可設定為支援拓撲感知資源分配以最佳化作業效能。支援三種模式：三維環面（用於 Cray 系統）、階層樹狀（fat-tree/dragonfly）和區塊拓撲。設定 `TopologyPlugin=topology/tree` 和 `topology.conf` 定義交換器階層。使用者可用 `--switches` 選項指定最大葉交換器數。從 25.05 版本起支援多重拓撲和動態拓撲。

---

## 翻譯

### 概觀

Slurm 可設定為支援拓撲感知資源分配以最佳化作業效能。Slurm 支援多種運作模式：

| 模式 | 適用網路 | 外掛程式 |
|------|----------|----------|
| 三維環面 | Cray XT/XE | 內建 |
| 階層樹狀 | Fat-tree, Dragonfly | topology/tree |
| 區塊拓撲 | 階層區塊結構 | topology/block |

Slurm 的原生資源選擇模式是將節點視為一維陣列，作業以最佳擬合方式分配資源。

---

### 三維拓撲

某些較大的電腦依賴三維環面互連。Cray XT 和 XE 系統也有三維環面互連，但不要求作業在相鄰節點上執行。

在這些系統上，Slurm 只需要將網路上相近的資源分配給作業。Slurm 使用 [希爾伯特曲線](http://en.wikipedia.org/wiki/Hilbert_curve) 將節點從三維空間映射到一維空間，使原生最佳擬合演算法能夠為作業實現高度的局部性。

---

### 樹狀拓撲（階層網路）

Slurm 可設定為在階層網路上分配資源以最小化網路競爭。

#### 基本演算法

1. 識別階層中可滿足作業請求的最低層級交換器
2. 使用最佳擬合演算法在其底層葉交換器上分配資源

#### 啟用設定

```
TopologyPlugin=topology/tree
```

#### topology.conf 設定

網路拓撲資訊必須包含在 `topology.conf` 設定檔中。

**三層交換器範例**：
```
# topology.conf
# 交換器設定
SwitchName=s0 Nodes=tux[0-1]
SwitchName=s1 Nodes=tux[2-3]
SwitchName=s2 Nodes=tux[4-5]
SwitchName=s3 Nodes=tux[6-7]
SwitchName=s4 Switches=s[0-1]
SwitchName=s5 Switches=s[2-3]
SwitchName=s6 Switches=s[4-5]
```

**帶有 LinkSpeed 的範例**：
```
# topology.conf
SwitchName=s0 Nodes=tux[0-3]   LinkSpeed=900
SwitchName=s1 Nodes=tux[4-7]   LinkSpeed=900
SwitchName=s2 Nodes=tux[8-11]  LinkSpeed=900
SwitchName=s3 Nodes=tux[12-15] LinkSpeed=1800
SwitchName=s4 Switches=s[0-3]  LinkSpeed=1800
```

**簡化設定**（建議）：
```
# topology.conf
SwitchName=s0 Nodes=tux[0-3]
SwitchName=s1 Nodes=tux[4-7]
SwitchName=s2 Nodes=tux[8-11]
SwitchName=s3 Nodes=tux[12-15]
SwitchName=s4 Switches=s[0-3]
```

**注意**：
- **SwitchName** 值是任意的，僅用於記帳目的
- 葉交換器描述包含 **SwitchName** 和 **Nodes** 欄位
- 高層交換器描述包含 **SwitchName** 和 **Switches** 欄位
- 支援 Slurm 的主機清單表示式（如 "Nodes=tux[0-3,12,18-20]"）
- **LinkSpeed** 是選用的，表示連結的相對效能

#### Dragonfly 網路

```
TopologyPlugin=topology/tree
TopologyParam=dragonfly
```

如果單一作業無法完全放置在單一網路葉交換器內，作業將分散到盡可能多的葉交換器以最佳化作業的網路頻寬。

---

### 區塊拓撲

Slurm 可設定為在嚴格執行的階層區塊結構內分配資源。

```
TopologyPlugin=topology/block
```

#### 區塊概念

- **基礎區塊 (bblocks)**：在 topology.conf 中定義的基本、連續節點群組
- **聚合區塊**：由相鄰基礎區塊組合形成
- **BlockSizes**：定義階層各層級的特定可執行區塊大小

#### 分配演算法

1. 識別可滿足作業資源請求的最小區塊層級（由 BlockSizes 定義）
2. 選擇合適的「低層區塊」子集
3. 使用最佳擬合演算法從構成此子集的基礎區塊中分配資源

#### 區塊拓撲限制

| 限制 | 說明 |
|------|------|
| 節點範圍 | 使用 -N 指定範圍時，排程器需評估每個值，建議保持範圍小 |
| 請求特定節點 | -w/--nodelist 可能與區塊放置衝突，目前不支援 |
| 連續區塊 | 無法請求將作業放置在非相鄰區塊上 |

---

### 使用者選項

#### --switches 選項

用於 topology/tree 外掛程式，使用者可指定作業的最大葉交換器數和等待此最佳化設定的最長時間：

```bash
--switches=count[@time]
```

**範例**：
```bash
sbatch --switches=2@10:00 script.sh
```

系統管理員可使用 `SchedulerParameters` 的 `max_switch_wait` 選項限制任何作業等待此最佳化設定的最長時間。

#### 主機清單函數

| 函數 | 說明 |
|------|------|
| `block{blockX}` | 展開為指定區塊中的所有節點 |
| `switch{switchY}` | 展開為指定交換器中的所有節點 |
| `blockwith{nodeX}` | 展開為與指定節點相同區塊中的所有節點 |
| `switchwith{nodeY}` | 展開為與指定節點相同交換器中的所有節點 |

**範例**：
```bash
scontrol update node=block{b1} state=resume
sbatch --nodelist=blockwith{node0} -N 10 program
PartitionName=Block10 Nodes=block{block10} ...
```

---

### 環境變數

使用 topology/tree 外掛程式時，會設定兩個環境變數描述作業的網路拓撲：

| 變數 | 說明 |
|------|------|
| **SLURM_TOPOLOGY_ADDR** | 從系統頂層交換器到葉交換器再到節點名稱的網路交換器名稱，以句點分隔 |
| **SLURM_TOPOLOGY_ADDR_PATTERN** | SLURM_TOPOLOGY_ADDR 中列出的元件類型（"switch" 或 "node"），以句點分隔 |

**注意**：這些環境變數在每個節點上啟動的任務中包含不同的資料。

---

### 多重拓撲

Slurm 25.05 引入了使用 `topology.yaml` 設定檔定義多個網路拓撲的功能。

每個分割區可透過在其設定行中指定 **Topology** 來設定使用特定拓撲。如果未明確指定分割區的拓撲，Slurm 將預設使用 cluster_default 拓撲。

---

### 動態拓撲

節點可動態新增到拓撲或從拓撲中移除。

#### 使用 scontrol

```bash
# 建立節點並指定拓撲
scontrol create NodeName=d[1-100] ... Topology=topo-switch:s1,topo-block:b1

# 更新節點拓撲
scontrol update NodeName=d[1-2] Topology=topo-switch:s2,topo-block:b2

# 從所有拓撲中移除節點
scontrol update NodeName=d100 Topology=
```

#### 使用 slurmd --conf

```bash
# 動態節點
slurmd -Z --conf "... Topology=topo-switch:s1,topo-block:b1"

# 使用預設拓撲
slurmd -Z --conf "... Topology=default:b1"

# 雲端節點（省略 -Z）
slurmd --conf "Topology=topo-cloud:s1"
```

**注意**：
- topology.conf 中定義的拓撲始終名為 "default"
- **拓撲單元**是區塊名稱或葉交換器名稱
- 可提供中間交換器名稱（以 ':' 分隔），如有需要會自動建立

---

### 設定產生器

以下獨立維護的工具可能有助於為某些交換器類型產生 topology.conf 檔案：

| 交換器類型 | 工具 | 連結 |
|------------|------|------|
| Infiniband | slurmibtopology | github.com/OleHolmNielsen/Slurm_tools/tree/master/slurmibtopology |
| Omni-Path (OPA) | opa2slurm | gitlab.com/jtfrey/opa2slurm |
| AWS EFA | ec2-topology | github.com/aws-samples/ec2-topology-aware-for-slurm |

---

## 說明

### 拓撲外掛程式比較

| 特性 | topology/tree | topology/block |
|------|---------------|----------------|
| 適用網路 | 階層式（fat-tree、dragonfly）| 嚴格階層區塊 |
| 分配策略 | 最小化網路跳躍 | 最小化碎片化 |
| 彈性 | 較高 | 較低 |
| 使用場景 | 一般 HPC | 特定架構 |

### 拓撲階層概念

```
頂層交換器
├── 中間交換器 1
│   ├── 葉交換器 A (nodes 1-4)
│   └── 葉交換器 B (nodes 5-8)
└── 中間交換器 2
    ├── 葉交換器 C (nodes 9-12)
    └── 葉交換器 D (nodes 13-16)
```

---

## 實務範例

### 基本樹狀拓撲設定

**slurm.conf：**
```
TopologyPlugin=topology/tree
```

**topology.conf：**
```
# 葉交換器
SwitchName=leaf1 Nodes=node[001-032]
SwitchName=leaf2 Nodes=node[033-064]
SwitchName=leaf3 Nodes=node[065-096]
SwitchName=leaf4 Nodes=node[097-128]

# 中間交換器
SwitchName=spine1 Switches=leaf[1-2]
SwitchName=spine2 Switches=leaf[3-4]

# 頂層交換器
SwitchName=core Switches=spine[1-2]
```

### Dragonfly 網路設定

**slurm.conf：**
```
TopologyPlugin=topology/tree
TopologyParam=dragonfly
```

### 區塊拓撲設定

**slurm.conf：**
```
TopologyPlugin=topology/block
```

**topology.conf：**
```
BlockSizes=16,64,256
BblockName=bb[1-16] Nodes=node[001-016]
BblockName=bb[17-32] Nodes=node[017-032]
...
```

### 使用 --switches 選項

```bash
# 限制使用最多 2 個葉交換器，等待最多 10 分鐘
sbatch --switches=2@10:00 mpi_job.sh

# 查看作業的交換器需求
squeue -o "%.18i %.9P %.8j %.8u %.8T %.10M %.9l %.6D %S"
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 列出所有交換器連線 | 簡化設定，只列葉交換器和一個頂層交換器 |
| 節點在無共同父交換器的交換器上 | 將不同網路段的節點放入不同分割區 |
| 忽略 Weight 與拓撲的關係 | 如果定義了 Weight，它會覆蓋基於拓撲的資源選擇 |
| 使用 -w 請求特定節點（區塊拓撲）| 區塊拓撲不支援，使用 -x 排除節點 |

### 效能考量

1. **簡化 topology.conf**：
   - 列出所有交換器連線會減慢排程
   - 通常列出葉交換器加一個頂層交換器即可

2. **分離網路段**：
   - 不同網路段的節點應放入不同分割區
   - 避免作業請求無法通訊的節點

3. **等待時間設定**：
   - 使用 `max_switch_wait` 限制最長等待時間
   - 平衡資源利用率和網路最佳化

---

## 快速參考

### slurm.conf 設定

```
# 樹狀拓撲
TopologyPlugin=topology/tree

# Dragonfly 網路
TopologyPlugin=topology/tree
TopologyParam=dragonfly

# 區塊拓撲
TopologyPlugin=topology/block

# 可選：允許跨無共同父交換器
TopologyParam=TopoOptional
```

### topology.conf 語法

| 欄位 | 說明 |
|------|------|
| SwitchName | 交換器名稱（任意） |
| Nodes | 連接到葉交換器的節點 |
| Switches | 子交換器清單 |
| LinkSpeed | 連結相對效能（選用） |

### 作業選項

| 選項 | 說明 |
|------|------|
| `--switches=N` | 最大葉交換器數 |
| `--switches=N@time` | 含等待時間 |

### 相關文件

- [topology.conf](topology.conf.html) - 拓撲設定檔
- [topology.yaml](topology.yaml.html) - 多重拓撲設定
- [slurm.conf](slurm.conf.html) - 主要設定檔
- [排程設定](sched_config.md) - 排程器設定
