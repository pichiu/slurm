# Slurm 原始碼樹狀分析

> 產生日期：2025-12-17 | 掃描等級：完整掃描

## 專案概覽

- **專案名稱**：Slurm 工作負載管理器
- **版本**：26.05.0-0rc1
- **儲存庫類型**：單體式
- **主要語言**：C (C99)
- **建置系統**：GNU Autotools

---

## 目錄結構

```
slurm/
├── src/                          # 主要原始碼（679 個 .c 檔案，419 個 .h 檔案）
│   ├── api/                      # Slurm C API 函式庫
│   ├── common/                   # 共用工具和資料結構
│   ├── conmgr/                   # 連線管理器
│   ├── curl/                     # HTTP/CURL 工具
│   ├── database/                 # 資料庫抽象層（MySQL）
│   ├── interfaces/               # 外掛介面定義
│   ├── lua/                      # Lua 腳本整合
│   │
│   ├── slurmctld/                # 中央控制守護程式 [守護程式]
│   ├── slurmd/                   # 節點守護程式 [守護程式]
│   │   ├── slurmd/               # 主要 slurmd 程式碼
│   │   ├── slurmstepd/           # 作業步驟管理器
│   │   └── common/               # 共用 slurmd 元件
│   ├── slurmdbd/                 # 資料庫守護程式 [守護程式]
│   ├── slurmrestd/               # REST API 守護程式 [守護程式]
│   │   └── plugins/              # REST API 外掛
│   │       ├── auth/             # JWT、本機驗證
│   │       └── openapi/          # OpenAPI 端點處理器
│   ├── sackd/                    # 記帳保活守護程式 [守護程式]
│   │
│   ├── sbatch/                   # 批次作業提交 [CLI]
│   ├── srun/                     # 平行作業啟動器 [CLI]
│   ├── salloc/                   # 互動式分配 [CLI]
│   ├── sattach/                  # 作業附加 [CLI]
│   ├── scancel/                  # 作業取消 [CLI]
│   ├── sbcast/                   # 檔案廣播 [CLI]
│   ├── squeue/                   # 作業佇列顯示 [CLI]
│   ├── sinfo/                    # 叢集資訊顯示 [CLI]
│   ├── sstat/                    # 執行中作業統計 [CLI]
│   ├── sacct/                    # 記帳報告 [CLI]
│   ├── sdiag/                    # 診斷 [CLI]
│   ├── sprio/                    # 優先級顯示 [CLI]
│   ├── scontrol/                 # 管理工具 [CLI]
│   ├── sacctmgr/                 # 記帳管理 [CLI]
│   ├── sreport/                  # 記帳報告 [CLI]
│   ├── sshare/                   # 公平共用資訊 [CLI]
│   ├── strigger/                 # 事件觸發器 [CLI]
│   ├── scrontab/                 # Cron 排程 [CLI]
│   ├── scrun/                    # 容器啟動器 [CLI]
│   ├── stepmgr/                  # 步驟管理 [CLI]
│   ├── sview/                    # GUI 監控 [CLI]
│   ├── bcast/                    # 廣播函式庫
│   │
│   └── plugins/                  # 外掛系統（38 種類別）
│       ├── accounting_storage/   # 資料庫後端
│       ├── acct_gather_*/        # 記帳收集器
│       ├── auth/                 # 驗證（munge、jwt、none、slurm）
│       ├── burst_buffer/         # 突發緩衝區管理
│       ├── cgroup/               # Cgroup v1/v2 支援
│       ├── cli_filter/           # CLI 命令過濾
│       ├── cred/                 # 憑證管理
│       ├── data_parser/          # 資料解析外掛
│       ├── gpu/                  # GPU 管理（nvidia、amd、intel）
│       ├── gres/                 # 通用資源
│       ├── hash/                 # 雜湊演算法
│       ├── job_submit/           # 作業提交鉤子
│       ├── jobacct_gather/       # 作業記帳
│       ├── jobcomp/              # 作業完成日誌
│       ├── mcs/                  # 多類別安全
│       ├── mpi/                  # MPI 支援（pmix、pmi2）
│       ├── node_features/        # 節點功能管理
│       ├── preempt/              # 作業搶佔
│       ├── priority/             # 優先級計算
│       ├── proctrack/            # 程序追蹤
│       ├── sched/                # 排程器（builtin、backfill）
│       ├── select/               # 資源選擇（linear、cons_tres）
│       ├── serializer/           # 資料序列化
│       ├── switch/               # 網路交換器支援
│       ├── task/                 # 任務管理
│       ├── tls/                  # TLS 加密
│       └── topology/             # 叢集拓樸
│
├── slurm/                        # 公開標頭檔
│   ├── slurm.h                   # 主要 API 標頭
│   ├── slurmdb.h                 # 記帳資料庫 API
│   ├── slurm_errno.h             # 錯誤代碼
│   ├── spank.h                   # SPANK 外掛 API
│   └── pmi.h                     # PMI 介面
│
├── doc/                          # 文件
│   ├── html/                     # 91 個 HTML 文件頁面
│   └── man/                      # Man 手冊頁面
│       ├── man1/                 # 22 個命令手冊
│       ├── man5/                 # 15 個設定檔手冊
│       └── man8/                 # 7 個管理手冊
│
├── etc/                          # 設定範例
│   └── slurm.conf.example        # 範例設定
│
├── testsuite/                    # 測試套件
│   ├── expect/                   # 基於 Expect 的功能測試
│   ├── python/                   # Python 測試
│   └── slurm_unit/               # 單元測試（Check 框架）
│
├── contribs/                     # 貢獻的模組
│   ├── lua/                      # Lua 綁定
│   ├── perlapi/                  # Perl API
│   ├── pam/                      # PAM 模組
│   ├── pam_slurm_adopt/          # PAM 採用模組
│   ├── pmi/                      # PMI 函式庫
│   ├── pmi2/                     # PMI2 函式庫
│   ├── torque/                   # Torque 相容性
│   └── openlava/                 # OpenLava 相容性
│
├── auxdir/                       # Autotools 支援檔案
│   ├── *.m4                      # Autoconf 巨集
│   └── slurm.m4                  # Slurm 專用巨集
│
├── debian/                       # Debian 套件
│
├── tools/                        # 建置和工具腳本
│
├── configure.ac                  # Autoconf 設定
├── Makefile.am                   # Automake 範本
├── slurm.spec                    # RPM 規格檔
├── META                          # 版本元資料
├── README.md                     # 專案說明
├── INSTALL                       # 安裝說明
├── CONTRIBUTING.md               # 貢獻指南
├── COPYING                       # GPL 授權
└── CHANGELOG/                    # 版本變更日誌
```

---

## 關鍵目錄

### 1. 核心守護程式（`src/slurmctld/`、`src/slurmd/`、`src/slurmdbd/`、`src/slurmrestd/`）

| 目錄 | 進入點 | 用途 |
|------|--------|------|
| `src/slurmctld/` | `controller.c:main()` | 中央控制器 - 作業排程、資源管理 |
| `src/slurmd/slurmd/` | `slurmd.c:main()` | 節點守護程式 - 本機作業執行 |
| `src/slurmd/slurmstepd/` | `slurmstepd.c:main()` | 作業步驟管理器 - 任務執行 |
| `src/slurmdbd/` | `slurmdbd.c:main()` | 資料庫守護程式 - 記帳 |
| `src/slurmrestd/` | `slurmrestd.c:main()` | REST API 守護程式 - HTTP 介面 |
| `src/sackd/` | `sackd.c:main()` | 記帳保活守護程式 |

### 2. CLI 工具（`src/s*/`）

| 工具 | 檔案 | 用途 |
|------|------|------|
| sbatch | `src/sbatch/sbatch.c` | 提交批次作業 |
| srun | `src/srun/srun.c` | 執行平行作業 |
| salloc | `src/salloc/salloc.c` | 互動式分配 |
| squeue | `src/squeue/squeue.c` | 檢視作業佇列 |
| sinfo | `src/sinfo/sinfo.c` | 檢視叢集資訊 |
| scontrol | `src/scontrol/scontrol.c` | 管理控制 |
| sacct | `src/sacct/sacct.c` | 記帳報告 |
| scancel | `src/scancel/scancel.c` | 取消作業 |

### 3. 共用函式庫（`src/common/`）

關鍵檔案：
- `job_record.h/c` - 作業資料結構
- `node_conf.h/c` - 節點設定
- `part_record.h/c` - 分割區管理
- `slurm_protocol_*.h/c` - RPC 協定
- `pack.h/c` - 序列化
- `log.h/c` - 日誌
- `xmalloc.h/c` - 記憶體管理

### 4. 外掛系統（`src/plugins/`）

38 種外掛類別，超過 100 種實作。主要類別：

| 類別 | 實作 | 用途 |
|------|------|------|
| `auth/` | jwt、munge、none、slurm | 驗證 |
| `select/` | linear、cons_tres | 資源選擇 |
| `sched/` | builtin、backfill | 作業排程 |
| `accounting_storage/` | mysql、slurmdbd、ctld_relay | 記帳儲存 |
| `cgroup/` | v1、v2 | 資源隔離 |

---

## 關鍵檔案位置

### 設定檔
- `etc/slurm.conf.example` - 主要設定範本
- `etc/slurmdbd.conf.example` - 資料庫守護程式設定
- `etc/cgroup.conf.example` - Cgroup 設定

### 公開標頭
- `slurm/slurm.h` - 主要 Slurm API
- `slurm/slurmdb.h` - 記帳 API
- `slurm/spank.h` - SPANK 外掛 API

### 進入點
- `src/slurmctld/controller.c` - 控制器守護程式
- `src/slurmd/slurmd/slurmd.c` - 節點守護程式
- `src/slurmrestd/slurmrestd.c` - REST 守護程式

---

## 統計資料

| 指標 | 數量 |
|------|------|
| C 原始碼檔案 (.c) | 679 |
| 標頭檔 (.h) | 419 |
| 外掛類別 | 38 |
| CLI 工具 | 19 |
| 守護程式 | 6 |
| HTML 文件頁面 | 91 |
| Man 手冊頁面 | 44 |
| 預估程式碼行數 | 500,000+ |
