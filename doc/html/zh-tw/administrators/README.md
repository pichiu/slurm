# Slurm 管理員指南 (Administrator Guide)

本目錄包含 Slurm 系統管理員相關文件的繁體中文翻譯。

## 文件列表

### 入門與概觀

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [quickstart_admin.md](quickstart_admin.md) | quickstart_admin.shtml | 管理員快速入門指南 |
| [overview.md](overview.md) | overview.shtml | Slurm 工作負載管理器概觀 |
| [upgrades.md](upgrades.md) | upgrades.shtml | 升級指南 |
| [big_sys.md](big_sys.md) | big_sys.shtml | 大型叢集管理指南 |
| [troubleshoot.md](troubleshoot.md) | troubleshoot.shtml | 故障排除指南 |

### 計費與資源管理

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [accounting.md](accounting.md) | accounting.shtml | 計費與資源使用追蹤 |
| [resource_limits.md](resource_limits.md) | resource_limits.shtml | 資源限制設定 |
| [qos.md](qos.md) | qos.shtml | 服務品質 (QoS) 設定 |
| [licenses.md](licenses.md) | licenses.shtml | 授權管理 |
| [wckey.md](wckey.md) | wckey.shtml | WCKey 管理 |
| [tres.md](tres.md) | tres.shtml | 可追蹤資源 (TRES) |

### 排程與優先順序

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [sched_config.md](sched_config.md) | sched_config.shtml | 排程設定指南 |
| [priority_multifactor.md](priority_multifactor.md) | priority_multifactor.shtml | 多因子優先順序設定 |
| [priority_multifactor3.md](priority_multifactor3.md) | priority_multifactor3.shtml | 深度無關公平分享 |
| [classic_fair_share.md](classic_fair_share.md) | classic_fair_share.shtml | 傳統公平分享演算法 |
| [fair_tree.md](fair_tree.md) | fair_tree.shtml | Fair Tree 公平分享演算法 |
| [preempt.md](preempt.md) | preempt.shtml | 作業搶佔設定 |
| [gang_scheduling.md](gang_scheduling.md) | gang_scheduling.shtml | Gang 排程（時間片輪轉） |
| [reservations.md](reservations.md) | reservations.shtml | 資源預約系統 |

### 資源選擇與排程

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [cons_tres.md](cons_tres.md) | cons_tres.shtml | 可消耗資源設定 |
| [cons_tres_share.md](cons_tres_share.md) | cons_tres_share.shtml | 共享可消耗資源 |
| [core_spec.md](core_spec.md) | core_spec.shtml | 核心專用化 |
| [topology.md](topology.md) | topology.shtml | 網路拓撲指南 |
| [high_throughput.md](high_throughput.md) | high_throughput.shtml | 高吞吐量運算指南 |
| [hres.md](hres.md) | hres.shtml | 階層資源 (HRES) 排程（Beta）|

### 硬體資源

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [gres.md](gres.md) | gres.shtml | 通用資源 (GPU/GRES) 設定 |
| [cgroups.md](cgroups.md) | cgroups.shtml | Linux Cgroups 整合指南 |

### 網路設定

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [network.md](network.md) | network.shtml | 網路設定指南 |

### 認證與安全

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [authentication.md](authentication.md) | authentication.shtml | 認證外掛程式設定 |
| [jwt.md](jwt.md) | jwt.shtml | JWT 認證指南 |
| [pam_slurm_adopt.md](pam_slurm_adopt.md) | pam_slurm_adopt.shtml | PAM 模組設定 |
| [mcs.md](mcs.md) | mcs.shtml | 多類別安全 (MCS) 指南 |
| [nss_slurm.md](nss_slurm.md) | nss_slurm.shtml | NSS Slurm 外掛程式 |
| [user_permissions.md](user_permissions.md) | user_permissions.shtml | 使用者權限設定 |
| [selinux.md](selinux.md) | selinux.shtml | SELinux 設定 |

### REST API

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [rest_quickstart.md](rest_quickstart.md) | rest_quickstart.shtml | REST API 快速入門指南 |
| [rest.md](rest.md) | rest.shtml | REST API 詳細說明 |
| [rest_clients.md](rest_clients.md) | rest_clients.shtml | REST API 客戶端撰寫指南 |
| [openapi_release_notes.md](openapi_release_notes.md) | openapi_release_notes.shtml | OpenAPI 外掛程式發行說明 |

### 作業完成與分析

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [elasticsearch.md](elasticsearch.md) | elasticsearch.shtml | Elasticsearch 整合 |
| [jobcomp_kafka.md](jobcomp_kafka.md) | jobcomp_kafka.shtml | Kafka 作業完成外掛程式 |

### 多叢集與聯邦

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [federation.md](federation.md) | federation.shtml | 聯邦排程指南 |

### 節能與雲端

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [power_save.md](power_save.md) | power_save.shtml | 節能指南 / 雲端爆發 |
| [dynamic_nodes.md](dynamic_nodes.md) | dynamic_nodes.shtml | 動態節點管理 |
| [configless_slurm.md](configless_slurm.md) | configless_slurm.shtml | 無設定檔模式 |

### 容器與儲存

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [containers.md](containers.md) | containers.shtml | 容器支援指南 |
| [burst_buffer.md](burst_buffer.md) | burst_buffer.shtml | 突發緩衝指南 |

### 整合與外部系統

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [kubernetes.md](kubernetes.md) | kubernetes.shtml | Kubernetes 整合 |

### 腳本與自動化

| 文件 | 原始文件 | 說明 |
|------|----------|------|
| [prolog_epilog.md](prolog_epilog.md) | prolog_epilog.shtml | Prolog 和 Epilog 腳本指南 |

## 備註

### 自動產生文件

以下文件為自動產生的 API 參考文件，不適合翻譯：

- rest_api.shtml - REST API 方法和模型（自動產生的 OpenAPI 文件，超過 700KB）

## 翻譯進度

- ✅ 已完成：49 個文件
- ⏳ 進行中：持續新增

## 相關連結

- [返回主目錄](../README.md)
- [使用者指南](../users/README.md)
- [Slurm 官方文件](https://slurm.schedmd.com/documentation.html)
