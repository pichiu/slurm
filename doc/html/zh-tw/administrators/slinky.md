# Slurm Slinky — Slurm 與 Kubernetes 整合

## TL;DR

Slinky 是 SchedMD 推出的專案集合，讓 Slurm 與 Kubernetes（容器編排平台）能夠互通協作。核心元件包含 slurm-operator（在 Kubernetes 上運行 Slurm 叢集）與 slurm-bridge（以 Slurm 作為 Kubernetes 排程器）。完整文件與程式碼倉庫皆公開於 [slinky.schedmd.com](https://slinky.schedmd.com/) 及 GitHub/GitLab。

---

## 翻譯

### 概觀

[Slinky&trade;](https://www.schedmd.com/slinky/why-slinky/) 是 SchedMD 所推出的一系列專案，旨在實現 Slurm 與 [Kubernetes](https://kubernetes.io/)（容器編排平台）之間的互通性（Interoperability）。

---

### 連結

Slinky 的官方文件可在[此處](https://slinky.schedmd.com/)取得。

Slinky 的程式碼倉庫（Repository）已公開於 [GitHub](https://github.com/SlinkyProject) 和 [GitLab](https://gitlab.com/SchedMD/slinky)：

- [slurm-operator](https://github.com/SlinkyProject/slurm-operator)：在 Kubernetes 上執行 Slurm。以 Pod 的形式在 Kubernetes 上管理並擴展 Slurm 叢集。
- [slurm-bridge](https://github.com/SlinkyProject/slurm-bridge)：以 Slurm 作為 Kubernetes 排程器（Scheduler）。使用 Slurm 同時排程 Slurm 和 Kubernetes 工作負載。
- [slurm-client](https://github.com/SlinkyProject/slurm-client)：用於 Slurm REST 通訊的 Golang 函式庫，由各 Slinky 倉庫共用。
- [containers](https://github.com/SlinkyProject/containers)：用於建置 Slurm 容器映像檔（Container Image）的 Dockerfile。

公開的容器映像檔和 Helm Chart 成品（Artifact）可在[此處](https://github.com/orgs/SlinkyProject/packages)取得。

---

### 簡報資料

請注意，較舊的簡報可能包含過時資訊。

#### 2025 年簡報

- [Slurm Bridge：Kubernetes 中的 Slurm 排程超能力](https://slurm.schedmd.com/MISC25/Slurm_Bridge_KubeCon_25.pdf)，Alan Mutschelknaus 與 Tim Wickberg，SchedMD（KubeCon NA，2025 年 11 月）
- [Slurm Bridge](https://slurm.schedmd.com/MISC25/Slurm_Bridge_CNCF-Batch-20250729.pdf)，Alan Mutschelknaus、Skyler Malinowski 與 Marlow Warnicke，SchedMD（CNCF Batch Working Group，2025 年 7 月）
- [Slinky：Slurm 與 Kubernetes 之間缺失的橋樑](https://slurm.schedmd.com/MISC25/Slinky-CUG2025.pdf)，Skyler Malinowski、Alan Mutschelknaus、Marlow Warnicke 與 Tim Wickberg，SchedMD（CUG25，2025 年 5 月）
- [Slinky：Kubernetes 中的 Slurm，高效能 AI 與 HPC 工作負載管理](https://slurm.schedmd.com/MISC25/Slinky-KubeConEurope2025.pdf)，Tim Wickberg，SchedMD（KubeCon Europe，2025 年 4 月）

#### 2024 年簡報

- [Slinky：Slurm 與 Kubernetes 之間缺失的橋樑](https://slurm.schedmd.com/SC24/Slinky-CANOPIE.pdf)，Skyler Malinowski 與 Tim Wickberg，SchedMD（SC24，2024 年 11 月）
- [Slinky — Slurm Operator（運算子）](https://slurm.schedmd.com/SLUG24/Slinky-Slurm-Operator.pdf)，Skyler Malinowski、Alan Mutschelknaus 與 Marlow Warnicke，SchedMD（SLUG24，2024 年 9 月）
- [Slinky — Slurm Bridge（橋接器）](https://slurm.schedmd.com/SLUG24/Slinky-Slurm-Bridge.pdf)，Skyler Malinowski、Alan Mutschelknaus 與 Marlow Warnicke，SchedMD（SLUG24，2024 年 9 月）
