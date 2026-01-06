# Slurm 開發者指南

> 產生日期：2025-12-17 | 適用對象：軟體開發人員

本指南提供 Slurm 的開發、建置、外掛開發和貢獻程式碼的完整說明。

---

## 目錄

1. [開發環境設定](#開發環境設定)
2. [建置系統](#建置系統)
3. [原始碼結構](#原始碼結構)
4. [核心概念](#核心概念)
5. [外掛開發](#外掛開發)
6. [API 開發](#api-開發)
7. [測試](#測試)
8. [除錯技巧](#除錯技巧)
9. [程式碼風格](#程式碼風格)
10. [貢獻流程](#貢獻流程)

---

## 開發環境設定

### 必要工具

```bash
# 建置工具
autoconf >= 2.59
automake
gcc 或 clang
make
libtool
pkg-config

# 核心函式庫
munge-devel
mysql-devel 或 mariadb-devel
readline-devel
python3
```

### 選用相依套件

```bash
# 進階功能
hwloc-devel       # 硬體拓樸感知
numactl-devel     # NUMA 支援
lua-devel >= 5.1  # Lua 腳本外掛
pam-devel         # PAM 整合
check >= 0.9.8    # 單元測試框架
pmix-devel        # PMIx MPI 支援
ucx-devel         # UCX 高效能網路
libcurl-devel     # REST API 支援
libjwt-devel      # JWT 驗證
```

### 快速開始

```bash
# 複製儲存庫
git clone https://github.com/SchedMD/slurm.git
cd slurm

# 開發建置
./configure --prefix=/usr/local \
            --enable-debug \
            --enable-developer \
            CFLAGS="-g -O0"

make -j$(nproc)
```

---

## 建置系統

### Configure 選項

| 選項 | 說明 |
|------|------|
| `--enable-debug` | 啟用除錯輸出和符號 |
| `--enable-developer` | 啟用額外的開發者檢查 |
| `--enable-multiple-slurmd` | 單機多 slurmd（測試用）|
| `--with-munge` | MUNGE 驗證支援 |
| `--with-jwt` | JWT 驗證支援 |
| `--with-slurmrestd` | 建置 REST API 守護程式 |
| `--with-pmix=PATH` | PMIx 路徑 |
| `--with-hwloc=PATH` | hwloc 路徑 |

### 建置目標

```bash
# 完整建置
make -j$(nproc)

# 只建置特定元件
make -C src/slurmctld

# 建置並安裝
make install

# 建置 RPM
make dist
rpmbuild -ta slurm-*.tar.bz2

# 清理
make clean
make distclean
```

---

## 原始碼結構

### 目錄配置

```
slurm/
├── src/
│   ├── api/              # 客戶端 API 函式庫
│   ├── common/           # 共用程式碼
│   ├── interfaces/       # 外掛介面定義
│   ├── plugins/          # 外掛實作
│   │   ├── auth/         # 驗證外掛
│   │   ├── sched/        # 排程外掛
│   │   ├── select/       # 資源選擇外掛
│   │   └── ...
│   ├── slurmctld/        # 控制器守護程式
│   ├── slurmd/           # 節點守護程式
│   │   ├── slurmd/       # 主程式
│   │   └── slurmstepd/   # 步驟守護程式
│   ├── slurmdbd/         # 資料庫守護程式
│   ├── slurmrestd/       # REST API 守護程式
│   └── s*/               # CLI 工具
├── slurm/                # 公開標頭檔
├── testsuite/            # 測試
└── doc/                  # 文件
```

### 關鍵檔案

| 檔案 | 說明 |
|------|------|
| `slurm/slurm.h` | 主要公開 API |
| `slurm/slurmdb.h` | 記帳 API |
| `src/common/job_record.h` | 作業資料結構 |
| `src/common/node_conf.h` | 節點資料結構 |
| `src/common/pack.c` | 序列化函式 |
| `src/common/slurm_protocol_defs.h` | RPC 訊息定義 |

---

## 核心概念

### 資料結構

**job_record_t** - 作業記錄：
```c
struct job_record {
    uint32_t job_id;
    char *account;
    uint32_t user_id;
    job_resources_t *job_resrcs;
    uint32_t job_state;
    time_t submit_time;
    time_t start_time;
    // ...
};
```

**node_record_t** - 節點記錄：
```c
struct node_record {
    char *name;
    uint16_t cpus;
    uint64_t real_memory;
    uint32_t node_state;
    List gres_list;
    // ...
};
```

### RPC 協定

Slurm 使用自訂二進位 RPC 協定：

```c
// 訊息類型定義於 slurm_protocol_defs.h
#define REQUEST_JOB_INFO           1001
#define RESPONSE_JOB_INFO          1002

// 訊息處理流程
// 1. pack 序列化請求
// 2. 發送至目標守護程式
// 3. unpack 反序列化
// 4. 處理請求
// 5. pack 序列化回應
// 6. 發送回應
```

### 序列化

使用 `pack.h` 進行資料序列化：

```c
#include "src/common/pack.h"

// 打包
buf_t *buffer = init_buf(1024);
pack32(value, buffer);
packstr(string, buffer);

// 解包
uint32_t value;
char *string = NULL;
safe_unpack32(&value, buffer);
safe_unpackstr(&string, buffer);
```

---

## 外掛開發

### 外掛結構

所有外掛必須實作以下符號：

```c
// myplugin.c
#include "slurm/slurm.h"

// 必要符號
const char plugin_name[] = "My Custom Plugin";
const char plugin_type[] = "select/myplugin";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

// 初始化（選用）
extern int init(void)
{
    // 初始化程式碼
    return SLURM_SUCCESS;
}

// 清理（選用）
extern int fini(void)
{
    // 清理程式碼
    return SLURM_SUCCESS;
}

// 外掛特定函式（根據外掛類型）
// ...
```

### 外掛類型

| 類型 | 介面 | 用途 |
|------|------|------|
| auth | `src/interfaces/auth.c` | 驗證 |
| select | `src/interfaces/select.c` | 資源選擇 |
| sched | `src/interfaces/sched.c` | 排程演算法 |
| job_submit | `src/interfaces/job_submit.c` | 作業提交鉤子 |
| gres | `src/interfaces/gres.c` | 通用資源 |
| topology | `src/interfaces/topology.c` | 網路拓樸 |

### 作業提交外掛範例

```c
// job_submit/myfilter.c
#include "slurm/slurm.h"
#include "src/slurmctld/slurmctld.h"

const char plugin_name[] = "Job Submit Filter";
const char plugin_type[] = "job_submit/myfilter";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

extern int job_submit(job_desc_msg_t *job_desc,
                      uint32_t submit_uid,
                      char **err_msg)
{
    // 檢查作業
    if (job_desc->time_limit > 86400) {
        *err_msg = xstrdup("Time limit too long");
        return ESLURM_INVALID_TIME_LIMIT;
    }

    // 修改作業
    if (job_desc->name == NULL) {
        job_desc->name = xstrdup("unnamed_job");
    }

    return SLURM_SUCCESS;
}

extern int job_modify(job_desc_msg_t *job_desc,
                      job_record_t *job_ptr,
                      uint32_t submit_uid,
                      char **err_msg)
{
    return SLURM_SUCCESS;
}
```

### 建置外掛

**樹內建置：**
```bash
# 在 src/plugins/job_submit/ 建立目錄
mkdir src/plugins/job_submit/myfilter

# 建立 Makefile.am
# 修改 configure.ac
# 重新執行 configure
```

**樹外建置：**
```bash
gcc -shared -fPIC -o job_submit_myfilter.so \
    myfilter.c \
    -I/usr/local/include/slurm
```

---

## API 開發

### C API 使用

```c
#include <slurm/slurm.h>
#include <slurm/slurmdb.h>

// 取得作業資訊
job_info_msg_t *job_info;
int rc = slurm_load_jobs(0, &job_info, SHOW_ALL);
if (rc == SLURM_SUCCESS) {
    for (int i = 0; i < job_info->record_count; i++) {
        printf("Job %u: %s\n",
               job_info->job_array[i].job_id,
               job_info->job_array[i].name);
    }
    slurm_free_job_info_msg(job_info);
}

// 提交作業
job_desc_msg_t job_desc;
slurm_init_job_desc_msg(&job_desc);
job_desc.script = "#!/bin/bash\nhostname";
job_desc.time_limit = 60;
job_desc.ntasks = 1;

submit_response_msg_t *resp;
rc = slurm_submit_batch_job(&job_desc, &resp);
if (rc == SLURM_SUCCESS) {
    printf("Submitted job %u\n", resp->job_id);
    slurm_free_submit_response_response_msg(resp);
}
```

### REST API

```bash
# 取得作業
curl -H "X-SLURM-USER-TOKEN: $TOKEN" \
     http://localhost:6820/slurm/v0.0.45/jobs/

# 提交作業
curl -X POST \
     -H "X-SLURM-USER-TOKEN: $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"job": {"name": "test", "ntasks": 1, "script": "#!/bin/bash\nhostname"}}' \
     http://localhost:6820/slurm/v0.0.45/job/submit
```

---

## 測試

### 單元測試

```bash
# 建置並執行單元測試
cd testsuite/slurm_unit
make check

# 執行特定測試
./common/slurmdb_pack-test
```

### 功能測試

```bash
# 需要運行中的叢集
cd testsuite/expect
./regression.py

# 執行特定測試
./test1.1
```

### Python 測試

```bash
cd testsuite/python
pytest

# 執行特定測試
pytest tests/test_jobs.py -v
```

---

## 除錯技巧

### 除錯建置

```bash
./configure --enable-debug --enable-developer CFLAGS="-g -O0"
```

### 前景執行

```bash
# 前景執行守護程式
slurmctld -D -vvvvv
slurmd -D -vvvvv
slurmdbd -D -vvvvv
```

### GDB 除錯

```bash
# 附加到運行中的程序
gdb -p $(pidof slurmctld)

# 從頭開始除錯
gdb --args slurmctld -D -vvvvv

# 常用 GDB 命令
(gdb) break job_mgr.c:1234
(gdb) continue
(gdb) print *job_ptr
(gdb) backtrace
```

### 日誌除錯

```bash
# 動態調整日誌等級
scontrol setdebug debug5

# 啟用特定除錯旗標
scontrol setdebugflags +Backfill,+SelectType

# 檢視日誌
tail -f /var/log/slurmctld.log
```

### 記憶體除錯

```bash
# 使用 Valgrind
valgrind --leak-check=full slurmctld -D

# 使用 AddressSanitizer
./configure CFLAGS="-fsanitize=address -g"
```

---

## 程式碼風格

### 格式規則

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

for (int i = 0; i < count; i++) {
        process(i);
}
```

### 命名慣例

```c
// 函式：小寫底線分隔
int slurm_load_jobs(time_t update_time, ...);

// 巨集：大寫底線分隔
#define SLURM_SUCCESS 0
#define MAX_JOB_COUNT 1000

// 類型：_t 後綴
typedef struct job_record job_record_t;
typedef uint32_t bitstr_t;

// 全域變數前綴
slurm_conf_t slurm_conf;
slurmdb_cluster_rec_t *working_cluster_rec;
```

### 記憶體管理

```c
// 使用 Slurm 記憶體函式
char *str = xstrdup("hello");
void *ptr = xmalloc(sizeof(job_record_t));
ptr = xrealloc(ptr, new_size);
xfree(ptr);

// 字串操作
char *result = NULL;
xstrfmtcat(result, "Job %u", job_id);
xstrcat(result, " completed");
```

### 錯誤處理

```c
// 錯誤訊息不要斷行
error("Job %u failed to allocate resources on node %s",
      job_id, node_name);

// 使用適當的日誌等級
fatal("Cannot continue without configuration");
error("Job submission failed");
info("Starting job %u", job_id);
debug("Processing step %u.%u", job_id, step_id);
debug2("Detailed trace information");
```

---

## 貢獻流程

### 提交修補程式

1. **建立修補程式**
   ```bash
   git format-patch -1 HEAD
   ```

2. **提交到問題追蹤**
   - 網站：https://support.schedmd.com/
   - 附上修補程式和說明

3. **不接受 GitHub Pull Request**

### 提交訊息格式

```
Changelog: 簡短說明變更內容

詳細說明做了什麼以及為什麼這樣做。
可以是多行說明。

Bug: 12345 (如果有相關的 bug 編號)
```

### 提交前檢查

- [ ] 程式碼遵循風格指南
- [ ] 沒有編譯警告
- [ ] 測試通過
- [ ] 文件已更新
- [ ] 提交訊息包含 Changelog
- [ ] 不包含產生的檔案

---

## 相關文件

- [專案概覽](./project-overview.md) - Slurm 系統概述
- [架構文件](./architecture.md) - 詳細架構說明
- [原始碼樹狀分析](./source-tree-analysis.md) - 原始碼結構
- [資料模型](./data-models.md) - 資料結構說明
- [API 契約](./api-contracts.md) - REST API 文件

---

## 進階外掛開發

以下為進階外掛開發主題，詳細說明請參考官方文件。

### CLI Filter 外掛

過濾或修改命令列工具的行為：

```c
// cli_filter/myfilter.c
#include "slurm/slurm.h"

const char plugin_name[] = "CLI Filter Plugin";
const char plugin_type[] = "cli_filter/myfilter";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

extern int cli_filter_setup_defaults(slurm_opt_t *opt, bool early)
{
    // 設定預設值
    return SLURM_SUCCESS;
}

extern int cli_filter_pre_submit(slurm_opt_t *opt, int offset)
{
    // 提交前過濾
    return SLURM_SUCCESS;
}

extern int cli_filter_post_submit(int offset, uint32_t job_id,
                                   uint32_t step_id)
{
    // 提交後處理
    return SLURM_SUCCESS;
}
```

📖 詳見：[CLI Filter 外掛程式設計指南](https://slurm.schedmd.com/cli_filter_plugins.html)

### PrEp（Prologue/Epilogue）外掛

作業生命週期鉤子：

```c
// prep/myprep.c
#include "slurm/slurm.h"

const char plugin_name[] = "PrEp Plugin";
const char plugin_type[] = "prep/myprep";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

extern int prep_p_prolog(job_env_t *job_env, slurm_cred_t *cred)
{
    // 作業開始前執行
    return SLURM_SUCCESS;
}

extern int prep_p_epilog(job_env_t *job_env, slurm_cred_t *cred)
{
    // 作業結束後執行
    return SLURM_SUCCESS;
}

extern int prep_p_prolog_slurmctld(job_record_t *job_ptr,
                                    job_env_t *job_env)
{
    // 控制器端作業開始前
    return SLURM_SUCCESS;
}

extern int prep_p_epilog_slurmctld(job_record_t *job_ptr,
                                    job_env_t *job_env)
{
    // 控制器端作業結束後
    return SLURM_SUCCESS;
}
```

📖 詳見：[PrEp 外掛程式設計指南](https://slurm.schedmd.com/prep_plugins.html)

### Site Factor（優先級）外掛

自訂作業優先級計算：

```c
// site_factor/myfactor.c
#include "slurm/slurm.h"

const char plugin_name[] = "Site Factor Plugin";
const char plugin_type[] = "site_factor/myfactor";
const uint32_t plugin_version = SLURM_VERSION_NUMBER;

extern void site_factor_p_set(job_record_t *job_ptr)
{
    // 計算並設定作業的 site_factor
    // 影響作業優先級
    job_ptr->site_factor = calculate_factor(job_ptr);
}

extern void site_factor_p_update(void)
{
    // 定期更新所有作業的 site_factor
}
```

📖 詳見：[Site Factor 外掛程式設計指南](https://slurm.schedmd.com/site_factor.html)

### 設計文件

深入了解 Slurm 內部設計：

| 主題 | 說明 |
|------|------|
| GRES 設計 | 通用資源處理架構 |
| 作業啟動設計 | 作業執行流程 |
| Select 外掛設計 | 資源選擇演算法 |

📖 詳見：
- [GRES 設計指南](https://slurm.schedmd.com/gres_design.html)
- [作業啟動設計指南](https://slurm.schedmd.com/job_launch.html)
- [Select 外掛設計指南](https://slurm.schedmd.com/select_design.html)

---

## 官方文件參考

以下連結指向 Slurm 官方文件，提供更詳細的說明：

### 開發入門
- [貢獻者指南](https://slurm.schedmd.com/contributor.html)
- [程式設計師指南](https://slurm.schedmd.com/programmer_guide.html)
- [應用程式設計介面（API）指南](https://slurm.schedmd.com/api.html)
- [新增檔案或外掛到 Slurm](https://slurm.schedmd.com/add.html)

### 設計文件
- [GRES 設計指南](https://slurm.schedmd.com/gres_design.html)
- [作業啟動設計指南](https://slurm.schedmd.com/job_launch.html)
- [Select 外掛設計指南](https://slurm.schedmd.com/select_design.html)

### 外掛開發
- [外掛程式設計師指南](https://slurm.schedmd.com/plugins.html)
- [CLI Filter 外掛程式設計指南](https://slurm.schedmd.com/cli_filter_plugins.html)
- [作業提交外掛程式設計指南](https://slurm.schedmd.com/job_submit_plugins.html)
- [PrEp 外掛程式設計指南](https://slurm.schedmd.com/prep_plugins.html)
- [Site Factor 外掛程式設計指南](https://slurm.schedmd.com/site_factor.html)

### REST API 開發
- [REST API 快速入門](https://slurm.schedmd.com/rest_quickstart.html)
- [REST API 詳細說明](https://slurm.schedmd.com/rest.html)
- [REST API 方法與模型](https://slurm.schedmd.com/rest_api.html)
- [OpenAPI 外掛發行說明](https://slurm.schedmd.com/openapi_release_notes.html)

---

## 外部資源

- **官方文件**：https://slurm.schedmd.com/
- **程式設計師指南**：https://slurm.schedmd.com/programmer_guide.html
- **API 參考**：https://slurm.schedmd.com/api.html
- **郵件清單**：slurm-users@lists.schedmd.com
- **問題追蹤**：https://support.schedmd.com/

---

## 相關文件

- [使用者指南](./user-guide.md) - 一般使用者文件
- [管理員指南](./admin-guide.md) - 系統管理員文件
