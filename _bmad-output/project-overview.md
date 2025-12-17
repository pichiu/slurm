# Slurm 工作負載管理器 - 專案概覽

> 產生日期：2025-12-17 | 版本：26.05.0-0rc1

## 什麼是 Slurm？

Slurm（Simple Linux Utility for Resource Management，簡易 Linux 資源管理工具）是一套開放原始碼、高度可擴展的叢集管理與作業排程系統，專為 Linux 叢集設計。最初由勞倫斯利佛摩國家實驗室開發，現由 SchedMD LLC 維護。

**主要功能：**
- 分配計算節點的獨佔或非獨佔存取權限
- 提供啟動、執行和監控平行作業的框架
- 透過作業佇列仲裁衝突的資源請求
- 支援從小型工作群組到數百萬核心的叢集

---

## 快速參考

| 屬性 | 值 |
|------|-----|
| **專案名稱** | Slurm 工作負載管理器 |
| **版本** | 26.05.0-0rc1 |
| **協定版本** | v45 |
| **主要語言** | C (C99) |
| **建置系統** | GNU Autotools |
| **授權** | GPLv2+ |
| **儲存庫類型** | 單體式 |
| **專案類型** | 後端、基礎設施、CLI |

### 技術堆疊

| 類別 | 技術 |
|------|------|
| 核心語言 | C (C99) |
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

## 架構摘要

```
使用者 → CLI 工具/REST API → slurmctld（控制器）
                                    ↓
                             slurmd（節點）
                                    ↓
                          slurmstepd（任務）
                                    ↓
                            slurmdbd → MySQL
```

### 核心元件

| 元件 | 用途 |
|------|------|
| **slurmctld** | 中央控制器 - 排程、資源分配 |
| **slurmd** | 節點守護程式 - 作業執行 |
| **slurmstepd** | 步驟管理器 - 任務管理 |
| **slurmdbd** | 資料庫守護程式 - 記帳 |
| **slurmrestd** | REST API 守護程式 - HTTP 介面 |

### CLI 工具

| 類別 | 工具 |
|------|------|
| 作業提交 | sbatch、srun、salloc |
| 作業控制 | scancel、sattach、sbcast |
| 監控 | squeue、sinfo、sstat、sacct、sdiag |
| 管理 | scontrol、sacctmgr、sreport |
| 公用程式 | sprio、sshare、strigger、scrontab |

---

## 專案統計

| 指標 | 數量 |
|------|------|
| C 原始碼檔案 | 679 |
| 標頭檔 | 419 |
| 外掛類別 | 38 |
| CLI 工具 | 19 |
| 守護程式 | 6 |
| HTML 文件頁面 | 91 |
| Man 手冊頁面 | 44 |
| 預估程式碼行數 | 500,000+ |

---

## 主要特色

### 排程

- 多種排程外掛（FIFO、backfill）
- 公平共用排程
- 多因素作業優先級
- 搶佔支援
- 保留與維護時段

### 資源管理

- CPU、記憶體、GPU 分配
- 通用資源（GRES）框架
- 可消耗資源追蹤
- 拓樸感知排程
- NUMA 感知放置

### 記帳

- 作業完成記錄
- 資源使用追蹤
- 使用者/帳戶/QoS 階層
- 公平共用計算
- 多叢集聯邦

### 安全性

- MUNGE 驗證
- JWT 權杖支援
- 作業憑證簽章
- 資源限制強制執行
- Cgroup 隔離

---

## 快速入門

### 使用者

```bash
# 提交批次作業
sbatch myjob.sh

# 執行互動式作業
srun --pty bash

# 檢查作業佇列
squeue -u $USER

# 檢查叢集狀態
sinfo
```

### 管理員

```bash
# 設定：編輯 /etc/slurm/slurm.conf
# 啟動服務：
systemctl start slurmctld
systemctl start slurmd

# 檢查診斷
sdiag
scontrol show config
```

### 開發者

```bash
# 從原始碼建置
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
```

---

## 文件連結

- **官方網站**：https://slurm.schedmd.com/
- **管理員指南**：https://slurm.schedmd.com/quickstart_admin.html
- **使用者指南**：https://slurm.schedmd.com/quickstart.html
- **API 參考**：https://slurm.schedmd.com/api.html
- **問題追蹤**：https://support.schedmd.com/

---

## 本文件集的相關檔案

- [架構文件](./architecture.md) - 系統架構詳細說明
- [原始碼樹狀分析](./source-tree-analysis.md) - 程式碼組織
- [API 契約](./api-contracts.md) - REST API 端點
- [資料模型](./data-models.md) - 資料庫結構
- [開發指南](./development-guide.md) - 建置與貢獻
