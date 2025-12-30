# Slurm 繁體中文文件 (Traditional Chinese Documentation)

本目錄包含 Slurm 工作負載管理器文件的繁體中文翻譯。

## 目錄結構

```
zh-tw/
├── README.md                     # 本文件
├── users/                        # 使用者指南
│   ├── README.md
│   ├── quickstart.md             # 快速入門
│   ├── job_array.md              # 作業陣列
│   ├── heterogeneous_jobs.md     # 異質作業
│   ├── cpu_management.md         # CPU 管理
│   ├── mpi_guide.md              # MPI 指南
│   ├── mc_support.md             # 多核心/多執行緒支援
│   ├── multi_cluster.md          # 多叢集操作
│   ├── hdf5_profile_user_guide.md # HDF5 效能剖析
│   ├── job_reason_codes.md       # 作業等待原因代碼
│   ├── job_state_codes.md        # 作業狀態代碼
│   ├── job_exit_code.md          # 作業結束代碼
│   ├── resource_binding.md       # 資源綁定
│   ├── rosetta.md                # 工作負載管理器對照表
│   └── faq.md                    # 常見問題
└── administrators/               # 管理員指南
    ├── README.md
    ├── overview.md               # Slurm 概觀
    ├── accounting.md             # 計費與資源限制
    ├── cgroups.md                # Cgroups 指南
    ├── gres.md                   # 通用資源 (GRES)
    ├── preempt.md                # 搶佔設定
    ├── priority_multifactor.md   # 多因子優先順序
    ├── qos.md                    # 服務品質 (QoS)
    ├── reservations.md           # 預約系統
    ├── resource_limits.md        # 資源限制
    └── troubleshoot.md           # 故障排除
```

## 文件分類

### [使用者指南](users/README.md)

適合一般 Slurm 使用者的文件，包括：
- 快速入門與基本操作
- 作業提交與管理
- CPU 和資源綁定
- 效能剖析工具

### [管理員指南](administrators/README.md)

適合 Slurm 系統管理員的文件：

| 文件 | 說明 |
|------|------|
| [overview.md](administrators/overview.md) | Slurm 概觀 |
| [accounting.md](administrators/accounting.md) | 計費與資源限制 |
| [cgroups.md](administrators/cgroups.md) | Cgroups 指南 |
| [gres.md](administrators/gres.md) | 通用資源 (GRES/GPU) |
| [preempt.md](administrators/preempt.md) | 搶佔設定 |
| [priority_multifactor.md](administrators/priority_multifactor.md) | 多因子優先順序 |
| [qos.md](administrators/qos.md) | 服務品質 (QoS) |
| [reservations.md](administrators/reservations.md) | 預約系統 |
| [resource_limits.md](administrators/resource_limits.md) | 資源限制 |
| [troubleshoot.md](administrators/troubleshoot.md) | 故障排除 |

## 翻譯說明

### 翻譯原則

1. **台灣用語**：採用台灣繁體中文慣用詞彙
2. **技術術語**：首次出現時標註英文原文，如：分割區 (partition)
3. **程式碼**：保持原始格式，不翻譯程式碼和命令
4. **一致性**：相同術語在所有文件中使用一致的翻譯

### 文件結構

每個翻譯文件包含以下章節：

1. **TL;DR** - 簡短摘要
2. **翻譯** - 完整翻譯內容
3. **說明** - 補充說明與概念解釋
4. **實務範例** - 實際使用範例
5. **常見錯誤與建議** - 常見問題和最佳實務
6. **快速參考** - 快速查閱表格

### 術語對照表

| 英文 | 中文 |
|------|------|
| Partition | 分割區 |
| Node | 節點 |
| Job | 作業 |
| Task | 任務 |
| Step | 步驟 |
| Socket | 插槽 |
| Core | 核心 |
| Thread | 執行緒 |
| Queue | 佇列 |
| Preemption | 搶佔 |
| Reservation | 預約 |
| Accounting | 計費 |
| QoS | 服務品質 |
| GRES | 通用資源 |

## 相關資源

- [Slurm 官方文件](https://slurm.schedmd.com/documentation.html)
- [Slurm GitHub](https://github.com/SchedMD/slurm)
- [SchedMD](https://www.schedmd.com/)

## 貢獻

如發現翻譯錯誤或有改善建議，歡迎提交 Issue 或 Pull Request。

## 授權

本翻譯文件遵循 Slurm 原始文件的授權條款。
