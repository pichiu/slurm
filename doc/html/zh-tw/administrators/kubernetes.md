# Slurm Kubernetes 指南

## TL;DR

Kubernetes 正被引入 HPC 叢集用於部署編排和特定工作負載（如 AI/ML 推論）。Slurm 和 Kubernetes 的整合仍在發展中，目標是實現統一叢集和優化資源利用。詳細資訊請參閱 SchedMD 和社群的相關簡報。

---

## 翻譯

### 概觀

[Kubernetes](https://kubernetes.io/) 正被引入 HPC 叢集用於編排部署（例如軟體、基礎設施）和執行特定工作負載（例如 AI/ML 推論）。業界對於整合 Kubernetes 和 Slurm 有持續的興趣，以實現：

- 統一的叢集管理
- 優化的資源利用
- 發揮各系統優勢的工作流程

Slurm 和 Kubernetes 在設計上處理不同類型工作負載的方式可能會隨時間改變。此外，它們之間的互動方式也可能改變，開創新的可能性。這仍是一個持續發展的領域。

---

### 相關簡報

**注意**：較舊的簡報可能包含過時的資訊。

#### 2023 年簡報

| 簡報標題 | 講者 | 場合 |
|----------|------|------|
| [Slurm and/or/vs Kubernetes](https://slurm.schedmd.com/SC23/Slurm-and-or-vs-Kubernetes.pdf) | Tim Wickberg, SchedMD | SC23, November 2023 |
| [Never use Slurm HA again: Solve all your problems with Kubernetes](https://slurm.schedmd.com/SLUG23/NERSC-SLUG23.pdf) | Chris Samuel and Doug Jacobsen, NERSC | SLUG23, November 2023 |

---

## 說明

### Slurm vs Kubernetes 比較

```
┌─────────────────────────────────────────────────────────────┐
│                    工作負載類型                              │
├────────────────────────┬────────────────────────────────────┤
│       Slurm            │          Kubernetes                │
├────────────────────────┼────────────────────────────────────┤
│ HPC 批次作業           │ 微服務                              │
│ MPI 並行運算           │ 容器化應用                          │
│ 科學模擬               │ AI/ML 推論服務                      │
│ 大型平行計算           │ 基礎設施編排                        │
└────────────────────────┴────────────────────────────────────┘
```

### 整合架構概念

```
┌─────────────────────────────────────────────────────────────┐
│                    統一叢集                                  │
│  ┌──────────────────┐      ┌──────────────────┐            │
│  │      Slurm       │◄────►│    Kubernetes    │            │
│  │  (HPC 工作負載)   │      │  (服務/容器)      │            │
│  └────────┬─────────┘      └────────┬─────────┘            │
│           │                         │                       │
│           ▼                         ▼                       │
│  ┌──────────────────────────────────────────────┐          │
│  │              共享計算資源                      │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

---

## 實務範例

### 常見整合場景

#### 1. 分離式部署

```
# 傳統方式：分別管理
┌─────────────┐     ┌─────────────┐
│ Slurm 叢集   │     │ K8s 叢集    │
│ (計算節點)   │     │ (服務節點)   │
└─────────────┘     └─────────────┘
      獨立運作           獨立運作
```

#### 2. Kubernetes 管理 Slurm 基礎設施

```
# 使用 Kubernetes 部署 Slurm 控制服務
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slurmctld
spec:
  replicas: 2  # 高可用性
  template:
    spec:
      containers:
      - name: slurmctld
        image: slurm-controller:latest
```

#### 3. 混合工作負載

```
# 根據工作負載類型選擇系統
if workload.type == "batch_hpc":
    submit_to_slurm(workload)
elif workload.type == "inference_service":
    deploy_to_kubernetes(workload)
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 強制統一所有工作負載 | 根據工作負載特性選擇適當平台 |
| 忽視各系統優勢 | 了解 Slurm 和 K8s 的設計理念 |
| 過度複雜的整合 | 從簡單整合開始，逐步擴展 |

### 建議

1. **評估需求**：
   - 確定哪些工作負載適合 Slurm
   - 確定哪些工作負載適合 Kubernetes
   - 評估整合的實際需求

2. **保持更新**：
   - 這是快速發展的領域
   - 關注 SchedMD 和社群的最新發展
   - 參與 SLUG 等社群會議

3. **參考資源**：
   - 查看上述簡報了解最新趨勢
   - 關注 Slurm 和 Kubernetes 社群討論

---

## 快速參考

### Slurm 與 Kubernetes 適用場景

| 場景 | 建議平台 |
|------|----------|
| 批次 HPC 作業 | Slurm |
| MPI 並行計算 | Slurm |
| 長時間科學模擬 | Slurm |
| 容器化微服務 | Kubernetes |
| AI/ML 推論服務 | Kubernetes |
| 基礎設施自動化 | Kubernetes |

### 發展趨勢

| 趨勢 | 說明 |
|------|------|
| 資源共享 | 在同一叢集上運行兩個系統 |
| 工作流程整合 | 作業可跨平台協調 |
| 容器化 HPC | Slurm 對容器支援增強 |

### 相關文件

- [容器支援](containers.md) - Slurm 容器整合
- [動態節點](dynamic_nodes.md) - 動態資源管理
- [節能指南](power_save.md) - 雲端爆發整合

