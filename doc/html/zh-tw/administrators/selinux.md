# Slurm SELinux 指南

## TL;DR

自 Slurm 21.08 起支援為作業設定 SELinux 安全上下文（技術預覽）。作業提交命令會自動設定當前操作上下文，可用 `--context` 覆寫。必須透過 job_submit 外掛程式驗證上下文，否則不會設定或管理上下文。編譯時需 `--enable-selinux` 和 libselinux1 函式庫。

---

## 翻譯

### 概觀

從版本 21.08 開始，Slurm 包含對為作業設定 SELinux 上下文的支援，作為技術預覽。此實作在未來版本中可能會改變，且預設未啟用支援。

---

### 架構

啟用後，Slurm 作業提交命令 — salloc、sbatch 和 srun — 會自動設定一個欄位為當前操作上下文。此欄位可透過 `--context` 命令列選項覆寫。

**重要說明**：

- 此值可被終端使用者直接操控
- 站點特定的腳本需要驗證和控制對這些上下文的存取
- 目前 MUNGE（Slurm 用於安全識別叢集上使用者和主機的工具）不提供 SELinux 上下文欄位
- 因此沒有安全機制可將當前上下文傳送到 Slurm 控制器
- 作業提交時提供的上下文**必須**由在 slurmctld 中運行的 job_submit 外掛程式驗證

**注意**：如果沒有這樣的腳本，不會為使用者的作業設定或管理任何上下文。

---

### 安裝

#### 原始碼編譯

SELinux 支援預設停用，必須在 configure 時啟用。需要 libselinux1 函式庫和開發標頭檔來建置。

```bash
./configure --enable-selinux
```

---

### 設定

安裝支援 SELinux 的 Slurm 版本後，您需要啟用並建立一個 job_submit 外掛程式，在將 SELinux 上下文傳遞給 slurmctld 之前執行驗證。目前沒有可靠且安全的方式在內部取得/驗證上下文，因此您**必須**建立此腳本並在 job_submit 外掛程式中執行驗證。

#### 範例 job_submit 外掛程式

```lua
function slurm_job_submit(job_desc, part_list, submit_uid)
  if job_desc.req_context then
    local element = 0
    for str in string.gmatch(job_desc.req_context, "([^:]+)") do
      if element == 0 and str ~= "unconfined_u" then
        slurm.log_user("Error: invalid SELinux context")
        return slurm.ERROR
      elseif element == 1 and str ~= "unconfined_r" then
        slurm.log_user("Error: %s is not a valid SELinux role")
        return slurm.ERROR
      end
      element = element + 1
    end
    job_desc.selinux_context = job_desc.req_context
  else
    -- 如果未請求上下文，強制使用特定上下文
    job_desc.selinux_context = "unconfined_u:unconfined_r:slurm_t:s0"
  end
  return slurm.SUCCESS
end
```

**說明**：
- `job_desc.selinux_context` 根據 `job_desc.req_context` 的內容設定（如果被認為有效）
- `job_desc.selinux_context` 是設定將用於作業的上下文

---

### 初始測試

`id` 命令對於顯示使用者當前的上下文非常有用。作為測試以確保我們正在切換上下文，您可以使用 srun 進行快速測試。

```bash
# 預設上下文
mcmult@master:~$ srun id
uid=1000(mcmult) gid=1000(mcmult) groups=1000(mcmult),27(sudo) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# 指定上下文
mcmult@master:~$ srun --context=unconfined_u:unconfined_r:unconfined_t:s0 id
uid=1000(mcmult) gid=1000(mcmult) groups=1000(mcmult),27(sudo) context=unconfined_u:unconfined_r:unconfined_t:s0
```

---

### 計費

目前 Slurm 的計費中不支援追蹤 SELinux 上下文。這可能會隨著未來版本中支援的發展而改變。如果您需要追蹤 SELinux 上下文，可以在 job_submit 外掛程式中將其儲存在管理員備註欄位中，如下例所示。

#### 計費追蹤範例

```lua
function slurm_job_submit(job_desc, part_list, submit_uid)
  if job_desc.req_context then
    local element = 0
    for str in string.gmatch(job_desc.req_context, "([^:]+)") do
      if element == 0 and str ~= "unconfined_u" then
        slurm.log_user("Error: invalid SELinux context")
        return slurm.ERROR
      elseif element == 1 and str ~= "unconfined_r" then
        slurm.log_user("Error: %s is not a valid SELinux role")
        return slurm.ERROR
      end
      element = element + 1
    end
    job_desc.selinux_context = job_desc.req_context
  else
    -- 如果未請求上下文，強制使用特定上下文
    job_desc.selinux_context = "unconfined_u:unconfined_r:slurm_t:s0"
  end
  -- 將上下文儲存到管理員備註以供追蹤
  job_desc.admin_comment = "SELinuxContext=" .. job_desc.selinux_context
  return slurm.SUCCESS
end
```

注意在回傳前設定 `job_desc.admin_comment`。這會將管理員備註設定為顯示我們將嘗試為作業設定的上下文。

---

### 注意事項

如果您希望同時使用 pam_slurm_adopt 和 SELinux，請參閱 [pam_slurm_adopt](pam_slurm_adopt.md) 文件以取得如何讓這兩者一起運作的提示。請注意，當同時使用此功能和 pam_slurm_adopt 時，ssh 會話可能不會落在與作業相同的上下文中。

---

## 說明

### SELinux 上下文格式

```
user:role:type:level

範例：unconfined_u:unconfined_r:unconfined_t:s0
      │            │            │            │
      └─ 使用者    └─ 角色      └─ 類型      └─ 安全等級
```

### 作業上下文流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 使用者提交   │────▶│ job_submit  │────▶│  slurmctld  │
│ --context   │     │ 驗證外掛程式 │     │ 設定上下文   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │   slurmd    │
                                        │ 執行作業    │
                                        │ (指定上下文) │
                                        └─────────────┘
```

---

## 實務範例

### 編譯啟用 SELinux

```bash
# 安裝相依套件
yum install libselinux-devel

# 編譯 Slurm
./configure --prefix=/usr/local \
            --sysconfdir=/etc/slurm \
            --enable-selinux
make
make install
```

### 建立驗證外掛程式

```lua
-- /etc/slurm/job_submit.lua

-- 允許的上下文白名單
local allowed_contexts = {
    "unconfined_u:unconfined_r:slurm_t:s0",
    "system_u:system_r:slurm_t:s0",
    "user_u:user_r:slurm_t:s0"
}

function is_allowed(context)
    for _, allowed in ipairs(allowed_contexts) do
        if context == allowed then
            return true
        end
    end
    return false
end

function slurm_job_submit(job_desc, part_list, submit_uid)
    if job_desc.req_context then
        if is_allowed(job_desc.req_context) then
            job_desc.selinux_context = job_desc.req_context
        else
            slurm.log_user("Error: SELinux context not allowed")
            return slurm.ERROR
        end
    else
        -- 預設上下文
        job_desc.selinux_context = "unconfined_u:unconfined_r:slurm_t:s0"
    end
    return slurm.SUCCESS
end
```

### slurm.conf 設定

```
# 啟用 job_submit 外掛程式
JobSubmitPlugins=lua
```

### 測試作業提交

```bash
# 測試預設上下文
srun id

# 測試指定上下文
srun --context=unconfined_u:unconfined_r:slurm_t:s0 id

# 測試無效上下文（應被拒絕）
srun --context=invalid:context:here:s0 id
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未建立 job_submit 外掛程式 | 必須建立驗證外掛程式 |
| 編譯時未啟用 SELinux | 使用 --enable-selinux 重新編譯 |
| 信任使用者提供的上下文 | 在外掛程式中驗證所有上下文 |
| 忽略 pam_slurm_adopt 相容性 | 測試 SSH 會話上下文 |

### 建議

1. **安全考量**：
   - 永遠在 job_submit 外掛程式中驗證上下文
   - 使用白名單而非黑名單
   - 記錄所有上下文變更

2. **測試流程**：
   - 先在測試環境啟用
   - 使用 `id` 命令驗證上下文切換
   - 測試無效上下文被正確拒絕

3. **計費整合**：
   - 使用 admin_comment 追蹤上下文
   - 考慮定期報表需求

---

## 快速參考

### 命令選項

| 命令 | 選項 | 說明 |
|------|------|------|
| salloc | --context=CTX | 設定作業上下文 |
| sbatch | --context=CTX | 設定作業上下文 |
| srun | --context=CTX | 設定作業上下文 |

### 編譯選項

```bash
./configure --enable-selinux
```

### job_submit 變數

| 變數 | 說明 |
|------|------|
| job_desc.req_context | 使用者請求的上下文 |
| job_desc.selinux_context | 實際設定的上下文 |
| job_desc.admin_comment | 可用於追蹤 |

### 相關文件

- [PAM Slurm Adopt](pam_slurm_adopt.md) - PAM 模組設定
- [MCS 指南](mcs.md) - 多類別安全
- [認證](authentication.md) - 認證設定

