# Slurm 開發指南

> 產生日期：2025-12-17

## 前置需求

### 必要相依套件

```bash
# 建置工具
autoconf >= 2.59
automake
gcc（或 clang）
make
libtool
pkg-config

# 核心函式庫
munge-devel       # 驗證（建議）
mysql-devel       # 或 mariadb-devel（用於記帳）
readline-devel    # 互動式 CLI
python3           # 建置腳本
perl-devel        # Perl API

# 選用相依套件
hwloc-devel       # 硬體拓樸
numactl-devel     # NUMA 支援
lua-devel >= 5.1  # Lua 腳本
pam-devel         # PAM 支援
check >= 0.9.8    # 單元測試
pmix-devel        # PMIx MPI 支援
ucx-devel         # UCX 網路
libcurl-devel     # HTTP 支援
libjwt-devel      # JWT 驗證
systemd-devel     # Systemd 整合
```

### 平台支援

- **主要**：Linux（廣泛測試）
- **架構**：x86_64、ARM64、POWER
- **發行版**：RHEL/CentOS、Ubuntu/Debian、SUSE、Fedora

---

## 建置說明

### 標準建置

```bash
# 複製儲存庫
git clone https://github.com/SchedMD/slurm.git
cd slurm

# 設定（自動偵測功能）
./configure --prefix=/usr/local

# 建置
make -j$(nproc)

# 安裝（以 root 身分）
sudo make install
```

### 開發建置

```bash
# 啟用除錯符號並停用最佳化
./configure --prefix=/usr/local \
            --enable-debug \
            --enable-developer \
            CFLAGS="-g -O0"

make -j$(nproc)
```

### 設定選項

| 選項 | 說明 |
|------|------|
| `--enable-debug` | 啟用除錯輸出和符號 |
| `--enable-developer` | 啟用開發者模式檢查 |
| `--enable-multiple-slurmd` | 允許每節點多個 slurmd |
| `--with-munge` | 啟用 MUNGE 驗證 |
| `--with-jwt` | 啟用 JWT 驗證 |
| `--with-pmix=PATH` | PMIx 安裝路徑 |
| `--with-ucx=PATH` | UCX 安裝路徑 |
| `--with-hwloc=PATH` | hwloc 安裝路徑 |
| `--with-lua=PATH` | Lua 安裝路徑 |
| `--with-slurmrestd` | 建置 REST API 守護程式 |
| `--with-yaml` | 啟用 YAML 序列化器 |
| `--without-pam` | 停用 PAM 支援 |

### RPM 建置

```bash
# 建立壓縮檔
make dist

# 建置 RPM
rpmbuild -ta slurm-*.tar.bz2

# 帶選項
rpmbuild -ta slurm-*.tar.bz2 \
         --with slurmrestd \
         --with jwt \
         --with pmix
```

---

## 專案結構

```
slurm/
├── configure.ac      # Autoconf 輸入
├── Makefile.am       # Automake 範本
├── src/              # 原始碼
│   ├── api/          # 客戶端 API
│   ├── common/       # 共用函式庫
│   ├── slurmctld/    # 控制器守護程式
│   ├── slurmd/       # 節點守護程式
│   ├── slurmdbd/     # 資料庫守護程式
│   ├── slurmrestd/   # REST 守護程式
│   ├── s*/           # CLI 工具
│   └── plugins/      # 外掛系統
├── slurm/            # 公開標頭
├── doc/              # 文件
├── testsuite/        # 測試
└── contribs/         # 貢獻的模組
```

---

## 程式碼規範

Slurm 遵循 [Linux 核心程式碼風格](https://www.kernel.org/doc/html/latest/process/coding-style.html)，並有一些修改：

### 格式

- **縮排**：Tab（8 個空格寬）
- **行長度**：最多 80 字元
- **大括號**：K&R 風格

```c
if (condition) {
        do_something();
        do_more();
} else {
        do_other();
}
```

### 命名慣例

- 函式：`lowercase_with_underscores`
- 巨集：`UPPERCASE_WITH_UNDERSCORES`
- 類型：typedef 使用 `name_t` 後綴
- 全域前綴：`slurm_`、`slurmdb_`

### 註解

```c
/* 單行註解 */

/*
 * 多行註解
 * 在此繼續
 */

// C++ 風格也可接受
```

### 錯誤訊息

- 不要在句子中間斷開錯誤字串
- 在格式序列、逗號或句號處分割
- 便於 grep 搜尋

```c
// 好的
error("Job %u failed to allocate resources on node %s",
      job_id, node_name);

// 不好 - 在句子中間斷開
error("Job %u failed to allocate "
      "resources on node %s", job_id, node_name);
```

---

## 測試

### 單元測試（Check 框架）

```bash
# 建置並執行單元測試
cd testsuite/slurm_unit
make check
```

位置：`testsuite/slurm_unit/`

### 功能測試（Expect）

```bash
# 執行 Expect 測試（需要運行中的 Slurm 叢集）
cd testsuite/expect
./regression.py
```

位置：`testsuite/expect/`

### Python 測試

```bash
# 執行 Python 測試
cd testsuite/python
pytest
```

位置：`testsuite/python/`

---

## 除錯

### 啟用除錯輸出

```bash
# 在 slurm.conf 中
SlurmctldDebug=debug5
SlurmdDebug=debug5
```

### GDB 除錯

```bash
# 附加到運行中的守護程式
gdb -p $(pidof slurmctld)

# 從啟動開始除錯
gdb --args slurmctld -D -vvvvv
```

### 日誌位置

- slurmctld：`/var/log/slurmctld.log`
- slurmd：`/var/log/slurmd.log`
- slurmdbd：`/var/log/slurmdbd.log`

---

## 外掛開發

### 外掛結構

```c
// myplugin.c
#include "slurm/slurm.h"

const char plugin_name[] = "My Plugin";
const char plugin_type[] = "select/myplugin";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

// 外掛初始化
extern int init(void)
{
    return SLURM_SUCCESS;
}

// 外掛清理
extern int fini(void)
{
    return SLURM_SUCCESS;
}

// 外掛專用函式...
```

### 建置外掛

外掛會隨主要建置自動建置。對於樹外建置：

```bash
gcc -shared -fPIC -o myplugin.so myplugin.c \
    -I/usr/local/include/slurm
```

---

## 常見開發任務

### 新增 CLI 選項

1. 編輯 `src/s*/opts.c` - 新增選項解析
2. 更新 `doc/man/man1/` 中的 man 頁面
3. 在 `testsuite/` 中新增測試

### 新增 RPC

1. 在 `src/common/slurm_protocol_defs.h` 中定義訊息類型
2. 在 `src/common/slurm_protocol_pack.c` 中新增 pack/unpack
3. 在相關守護程式中新增處理器
4. 如需要則更新協定版本

### 新增外掛類型

1. 在 `src/interfaces/` 中定義介面
2. 在 `src/plugins/` 中建立外掛目錄
3. 新增至 `configure.ac` 和 `Makefile.am`
4. 在 `doc/html/` 中記錄文件

---

## 貢獻流程

1. **問題追蹤**：https://support.schedmd.com/
2. **不接受 Pull Request** - 透過問題追蹤提交修補程式
3. **修補程式格式**：使用 `git format-patch`
4. **目標分支**：
   - 新功能 → `master`
   - 錯誤修正 → 最近的穩定版本

### 提交訊息格式

```
Changelog: 變更的簡短說明

詳細說明做了什麼以及為什麼。
```

### 提交檢查清單

- [ ] 程式碼遵循風格指南
- [ ] 測試通過
- [ ] 文件已更新
- [ ] 提交訊息包含 Changelog 尾碼
- [ ] 修補程式可正常套用
- [ ] 不包含產生的檔案

---

## 資源

### 文件

- 官方：https://slurm.schedmd.com/
- 管理員指南：https://slurm.schedmd.com/quickstart_admin.html
- API 參考：https://slurm.schedmd.com/api.html

### 原始碼參考

| 主題 | 位置 |
|------|------|
| 主要標頭 | `slurm/slurm.h`、`slurm/slurmdb.h` |
| 協定 | `src/common/slurm_protocol_*.h` |
| 作業管理 | `src/slurmctld/job_mgr.c` |
| 節點管理 | `src/slurmctld/node_mgr.c` |
| 排程 | `src/slurmctld/job_scheduler.c` |
| 外掛 API | `src/interfaces/` |

### 社群

- 郵件清單：slurm-users@lists.schedmd.com
- 問題追蹤：https://support.schedmd.com/
