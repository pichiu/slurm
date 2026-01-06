# Slurm 工作負載管理器 - 文件索引

> 產生日期：2025-12-17 | 最後更新：2026-01-06 | 版本：26.05.0-0rc1

本文件集為 Slurm 工作負載管理器程式碼庫的 AI 輔助開發而產生，提供針對 LLM 上下文效率最佳化的完整參考資料。

---

## 快速導覽

| 我想要... | 前往 |
|-----------|------|
| 了解 Slurm 是什麼 | [專案概覽](./project-overview.md) |
| 學習系統架構 | [架構文件](./architecture/architecture.md) |
| 瀏覽原始碼結構 | [原始碼樹狀分析](./architecture/source-tree-analysis.md) |
| 使用 REST API | [API 契約](./architecture/api-contracts.md) |
| 了解資料結構 | [資料模型](./architecture/data-models.md) |
| 配置 Partition | [Partition 指南](./technical-reference/configuration/slurm-partition-guide.md) |
| 配置 slurm.conf | [slurm.conf 說明](./technical-reference/configuration/slurm-conf.md) |

---

## 文件結構

```
_bmad-output/
├── index.md                    # 本文件 - 主索引
├── project-overview.md         # 專案概覽
│
├── architecture/               # 架構相關
│   ├── architecture.md         # 整體架構
│   ├── data-models.md          # 資料模型
│   ├── api-contracts.md        # API 契約
│   └── source-tree-analysis.md # 原始碼結構
│
├── guides/                     # 使用指南
│   ├── user-guide.md           # 使用者指南
│   ├── admin-guide.md          # 管理員指南
│   └── developer-guide.md      # 開發者指南
│
└── technical-reference/        # 深度技術參考
    ├── configuration/          # 配置相關
    │   ├── slurm-conf.md       # slurm.conf 完整說明
    │   └── slurm-partition-guide.md  # Partition 配置指南
    │
    ├── authentication/         # 認證相關
    │   ├── slurm-auth-munge.md # 認證機制分析
    │   └── slurm-ldap-integration.md # LDAP 整合
    │
    └── internals/              # 內部實作
        └── scontrol-and-restapi-deep-dive.md
```

---

## 依角色分類的指南

| 文件 | 說明 | 適用對象 |
|------|------|----------|
| [使用者指南](./guides/user-guide.md) | 作業提交、監控、資源查詢 | 一般使用者 |
| [管理員指南](./guides/admin-guide.md) | 叢集配置、節點管理、帳戶管理 | 系統管理員 |
| [開發者指南](./guides/developer-guide.md) | 建置、外掛開發、貢獻程式碼 | 開發人員 |

---

## 架構文件

| 文件 | 說明 |
|------|------|
| [架構文件](./architecture/architecture.md) | 系統設計、守護程式、外掛、通訊協定 |
| [原始碼樹狀分析](./architecture/source-tree-analysis.md) | 目錄結構、關鍵檔案、統計資料 |
| [API 契約](./architecture/api-contracts.md) | slurmctld 與 slurmdbd 的 REST API 端點 |
| [資料模型](./architecture/data-models.md) | 核心資料結構與資料庫綱要 |

---

## 深度技術參考

### 配置相關

| 文件 | 說明 |
|------|------|
| [slurm.conf 完整說明](./technical-reference/configuration/slurm-conf.md) | 主配置檔所有參數詳解 |
| [Partition 配置指南](./technical-reference/configuration/slurm-partition-guide.md) | Partition 資料結構、配置參數與實務範例 |

### 認證相關

| 文件 | 說明 |
|------|------|
| [認證機制分析](./technical-reference/authentication/slurm-auth-munge.md) | auth/munge 與 auth/slurm 深入分析 |
| [LDAP 整合](./technical-reference/authentication/slurm-ldap-integration.md) | Slurm 與 LDAP 整合方式 |

### 內部實作

| 文件 | 說明 |
|------|------|
| [scontrol 與 REST API](./technical-reference/internals/scontrol-and-restapi-deep-dive.md) | scontrol CLI 與 slurmrestd 架構分析 |

---

## 狀態檔案

| 檔案 | 用途 |
|------|------|
| [project-scan-report.json](./project-scan-report.json) | 工作流程狀態與掃描結果 |

---

## 專案現有文件

### 官方文件（儲存庫內）

| 位置 | 內容 |
|------|------|
| `doc/html/` | 91 個 HTML 文件頁面 |
| `doc/man/` | 44 個 man 手冊頁面 |
| `README.md` | 專案介紹 |
| `INSTALL` | 安裝說明 |
| `CONTRIBUTING.md` | 貢獻指南 |
| `CHANGELOG.md` | 版本歷史 |
| `SECURITY.md` | 安全政策 |

### 外部資源

- **官方網站**：https://slurm.schedmd.com/
- **完整文件索引**：https://slurm.schedmd.com/documentation.html
- **管理員指南**：https://slurm.schedmd.com/quickstart_admin.html
- **使用者指南**：https://slurm.schedmd.com/quickstart.html
- **API 參考**：https://slurm.schedmd.com/api.html
- **問題追蹤**：https://support.schedmd.com/

### 依角色分類的官方文件

詳細的官方文件連結請參考各角色指南中的「官方文件參考」章節：

| 角色 | 本地指南 | 官方文件數量 |
|------|----------|--------------|
| 使用者 | [使用者指南](./guides/user-guide.md#官方文件參考) | 14+ 篇文件 |
| 管理員 | [管理員指南](./guides/admin-guide.md#官方文件參考) | 40+ 篇文件 |
| 開發者 | [開發者指南](./guides/developer-guide.md#官方文件參考) | 12+ 篇文件 |

---

## 專案摘要

### 分類

| 屬性 | 值 |
|------|-----|
| 儲存庫類型 | 單體式 (Monolith) |
| 主要類型 | 後端服務（基礎設施 + CLI） |
| 架構模式 | 分散式主從架構 |
| 主要語言 | C (C99) |
| 建置系統 | GNU Autotools |

### 關鍵統計

| 指標 | 數量 |
|------|------|
| C 原始碼檔案 | 679 |
| 標頭檔 | 419 |
| 外掛類別 | 38 |
| CLI 工具 | 19 |
| 守護程式 | 6 |
| 預估程式碼行數 | 500,000+ |

### 技術堆疊

| 類別 | 技術 |
|------|------|
| 語言 | C (C99) |
| 資料庫 | MySQL / MariaDB 5.0+ |
| 驗證 | MUNGE（預設）、JWT |
| 建置 | autoconf、automake、libtool |
| 測試 | Check、Expect、Pytest |
| 序列化 | 自訂二進位、JSON、YAML |

### 網路連接埠

| 服務 | 連接埠 |
|------|--------|
| slurmctld | 6817 |
| slurmd | 6818 |
| slurmdbd | 6819 |
| slurmrestd | 6820 |

---

## 架構概覽

```
使用者 --> CLI 工具/REST API --> slurmctld（控制器）
                                      |
                                slurmd（節點）
                                      |
                              slurmstepd（任務）
                                      |
                                slurmdbd --> MySQL
```

### 核心元件

| 元件 | 用途 | 進入點 |
|------|------|--------|
| slurmctld | 中央控制器 - 排程、資源分配 | `src/slurmctld/controller.c` |
| slurmd | 節點守護程式 - 作業執行 | `src/slurmd/slurmd/slurmd.c` |
| slurmstepd | 步驟管理器 - 任務管理 | `src/slurmd/slurmstepd/slurmstepd.c` |
| slurmdbd | 資料庫守護程式 - 記帳 | `src/slurmdbd/slurmdbd.c` |
| slurmrestd | REST API 守護程式 - HTTP 介面 | `src/slurmrestd/slurmrestd.c` |

### CLI 工具

| 類別 | 工具 |
|------|------|
| 作業提交 | sbatch、srun、salloc |
| 作業控制 | scancel、sattach、sbcast |
| 監控 | squeue、sinfo、sstat、sacct、sdiag |
| 管理 | scontrol、sacctmgr、sreport |
| 公用程式 | sprio、sshare、strigger、scrontab |

---

## 原始碼導覽

### 關鍵目錄

| 目錄 | 用途 |
|------|------|
| `src/slurmctld/` | 控制器守護程式 |
| `src/slurmd/` | 節點守護程式與步驟守護程式 |
| `src/slurmdbd/` | 資料庫守護程式 |
| `src/slurmrestd/` | REST API 守護程式 |
| `src/common/` | 共用函式庫 |
| `src/interfaces/` | 外掛介面 |
| `src/plugins/` | 38 種外掛類別 |
| `src/api/` | 客戶端 API |
| `slurm/` | 公開標頭檔 |

### 理解程式碼的關鍵檔案

| 檔案 | 用途 |
|------|------|
| `src/common/job_record.h` | 作業資料結構 |
| `src/common/node_conf.h` | 節點資料結構 |
| `src/common/part_record.h` | 分割區資料結構 |
| `slurm/slurm.h` | 主要公開 API |
| `slurm/slurmdb.h` | 記帳 API |
| `src/common/slurm_protocol_defs.h` | RPC 定義 |
| `src/common/pack.c` | 序列化 |

---

## 開發快速參考

### 建置

```bash
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
```

### 除錯建置

```bash
./configure --prefix=/usr/local \
            --enable-debug \
            --enable-developer \
            CFLAGS="-g -O0"
make -j$(nproc)
```

### 測試

```bash
# 單元測試
cd testsuite/slurm_unit && make check

# 功能測試（需要運行中的叢集）
cd testsuite/expect && ./regression.py

# Python 測試
cd testsuite/python && pytest
```

---

## 文件修訂歷史

| 日期 | 動作 | 備註 |
|------|------|------|
| 2026-01-06 | 目錄結構重整 | 新增分類目錄、新增 Partition 配置指南 |
| 2025-12-31 | 新增深度技術文件 | 認證機制、LDAP 整合、scontrol/REST API |
| 2025-12-17 | 初始產生 | 完整掃描完成 |

---

## AI 開發說明

本文件針對 AI 輔助開發最佳化：

1. **專案概覽** - 從這裡開始了解背景
2. **架構文件** - 修改前先了解系統設計
3. **原始碼樹狀分析** - 快速找到相關檔案
4. **資料模型** - 編寫程式碼前了解核心結構
5. **API 契約** - REST API 工作的參考
6. **開發者指南** - 遵循程式碼規範
7. **技術參考** - 深入了解特定主題

### 常見任務

| 任務 | 建議閱讀 |
|------|----------|
| 新增 CLI 選項 | [開發者指南](./guides/developer-guide.md) > 常見開發任務 |
| 新增 RPC | [架構文件](./architecture/architecture.md) > 通訊協定 |
| 建立新外掛 | [架構文件](./architecture/architecture.md) > 外掛架構 |
| 修改作業處理 | [資料模型](./architecture/data-models.md) > job_record_t |
| 使用 REST API | [API 契約](./architecture/api-contracts.md) |
| 配置 Partition | [Partition 指南](./technical-reference/configuration/slurm-partition-guide.md) |
| 配置認證 | [認證機制](./technical-reference/authentication/slurm-auth-munge.md) |
