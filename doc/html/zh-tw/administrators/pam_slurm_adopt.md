# Slurm pam_slurm_adopt 模組

## TL;DR

pam_slurm_adopt 防止使用者 SSH 登入沒有執行作業的節點，並將 SSH 連線追蹤納入作業的「外部」步驟以確保完整的作業清理。需要設定 `PrologFlags=contain` 和 task/cgroup 外掛程式。在 /etc/pam.d/sshd 中新增 `account required pam_slurm_adopt.so`。生產環境應設定 `action_adopt_failure=deny` 和 `action_generic_failure=deny`。

---

## 翻譯

### 概觀

此模組的目的是：
1. 防止使用者 SSH 登入他們沒有執行作業的節點
2. 追蹤 SSH 連線和任何其他衍生程序以進行計費
3. 確保作業完成時的完整清理

此模組透過判斷發起 SSH 連線的作業來實現。使用者的連線會被「納入」作業的「外部」步驟。當存取被拒絕時，使用者會收到相關的錯誤訊息。

---

### 安裝

#### 從原始碼安裝

在 Slurm 原始碼目錄中，進入 `./contribs/pam_slurm_adopt/` 並以 **root** 身份執行：

```bash
make && make install
```

這會將 pam_slurm_adopt.a、pam_slurm_adopt.la 和 pam_slurm_adopt.so 安裝到：
- Debian 系統：/lib/security/
- RedHat/SUSE 系統：/lib64/security/

**注意**：安裝位置不受 configure 的 --prefix 旗標影響；使用 `--with-pam_dir=PATH` 修改安裝位置。

#### RPM 安裝

slurm.spec 會建構 slurm-pam_slurm RPM。

#### DEB 安裝

debian 打包腳本會建構 slurm-smd-libpam-slurm-adopt 套件。

---

### Slurm 設定

**必要設定**：

| 設定 | 說明 |
|------|------|
| `PrologFlags=contain` | 設定「extern」步驟供 SSH 啟動的程序納入（**必須在使用此模組前設定**）|
| task/cgroup 外掛程式 | 必須啟用 |

**選用設定**：

| 設定 | 說明 |
|------|------|
| `LaunchParameters=ulimit_pam_adopt` | 在外部步驟納入的程序中設定 RLIMIT_RSS |

**注意**：slurm.conf 中的 **UsePAM** 選項與 pam_slurm_adopt 無關。

**警告**：沒有 `PrologFlags=contain` 選項啟動的作業沒有 extern 步驟，pam_slurm_adopt 將無法存取這些作業。

---

### SSH 設定

1. 確認 /etc/ssh/sshd_config 中 `UsePAM` 設為 `On`（預設應該是開啟的）

2. 確保只啟用支援的 **AuthenticationMethods**：
   - 支援：**publickey**、**password**
   - 不支援：**keyboard-interactive**（必須移除）

**警告**：如果不遵守此步驟，程序納入會中斷，SSH 會話會在作業結束後持續存在。

---

### PAM 設定

#### 基本設定

在 /etc/pam.d 的適當檔案（如 sshd）中新增以下行：

```
account    required      pam_slurm_adopt.so
```

**重要**：
- pam_slurm_adopt.so 應該是 account 堆疊中的最後一個 PAM 模組
- 包含的檔案（如 common-account）通常應該在 pam_slurm_adopt 之前包含

#### 建議的 sshd PAM 堆疊

```
account    required      pam_nologin.so
account    include       password-auth
...
-account    required      pam_slurm_adopt.so
```

**注意**：pam_slurm_adopt 前的 "-" 允許 PAM 在找不到 pam_slurm_adopt.so 時優雅地失敗（建議用於 NFS 等共享檔案系統）。

#### 必要條件

- 必須使用 task/cgroup 任務外掛程式和 proctrack/cgroup proctrack 外掛程式
- pam_systemd 模組會與 pam_slurm_adopt 衝突，需要在所有相關檔案中停用

#### 生產環境設定

```
-account    required      pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny
```

**警告**：預設設定是為了確保管理員在測試期間不會被鎖定。生產環境應將 `action_adopt_failure` 和 `action_generic_failure` 都設為 **deny**。

#### 使用 pam_systemd 功能

如果需要 pam_systemd 的使用者管理功能（如處理 /run/user/$UID），可以使用 prolog/epilog 腳本：

**Prolog：**
```bash
loginctl enable-linger $SLURM_JOB_USER
exit 0
```

**TaskProlog：**
```bash
echo "export XDG_RUNTIME_DIR=/run/user/$SLURM_JOB_UID"
echo "export XDG_SESSION_ID=$(</proc/self/sessionid)"
echo "export XDG_SESSION_TYPE=tty"
echo "export XDG_SESSION_CLASS=user"
```

**Epilog：**
```bash
# 僅在這是此使用者最後一個作業時停用 linger
O_P=0
for pid in $(scontrol listpids | awk -v jid=$SLURM_JOB_ID 'NR!=1 { if ($2 != jid && $1 != "-1"){print $1} }'); do
    ps --noheader -o euser p $pid | grep -q $SLURM_JOB_USER && O_P=1
done
if [ $O_P -eq 0 ]; then
    loginctl disable-linger $SLURM_JOB_USER
fi
exit 0
```

---

### 管理員存取設定

pam_slurm_adopt 始終允許 root 使用者存取。要允許其他管理員使用自己的帳戶存取，可以堆疊其他模組。

#### 使用 pam_access

**/etc/security/access.conf：**
```
+:(wheel):ALL
-:ALL:ALL
```

**方式一：管理員有作業時 SSH 會話納入作業**
```
account    sufficient    pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny
account    required      pam_access.so
```

**方式二：管理員完全繞過 pam_slurm_adopt**
```
account    sufficient    pam_access.so
account    required      pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny
```

#### 使用 pam_listfile

```
account    sufficient    pam_listfile.so item=user sense=allow onerr=fail file=/path/to/allowed_users_file
account    required      pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny
```

#### 使用群組檢查和 pam_systemd

```
account    sufficient                    pam_listfile.so item=group sense=allow onerr=fail file=/path/to/allowed_groups_file
-account   required                      pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny

session    [default=1 success=ignore]    pam_listfile.so item=group sense=allow onerr=fail file=/etc/groupfile
-session   optional                      pam_systemd.so
```

---

### 模組選項

將選項新增到 /etc/pam.d/ 中適當檔案的 pam_slurm_adopt 行尾：

```
account sufficient pam_slurm_adopt.so optionname=optionvalue
```

#### action_no_jobs

使用者在節點上沒有作業時執行的動作。

| 值 | 說明 |
|----|------|
| **ignore** | 不做任何事，繼續到下一個 PAM 模組 |
| **deny**（預設）| 拒絕連線 |

#### action_unknown

使用者在節點上有多個作業且 RPC 無法定位來源作業時執行的動作。

| 值 | 說明 |
|----|------|
| **newest**（預設）| cgroup/v1：選擇最新的作業（根據 cgroup mtime）；cgroup/v2：選擇最大 ID 的作業 |
| **allow** | 允許連線但不納入 |
| **deny** | 拒絕連線 |

#### action_adopt_failure

程序無法納入任何作業時執行的動作。

| 值 | 說明 |
|----|------|
| **allow**（預設）| 允許連線但不納入（**警告**：僅建議測試用）|
| **deny** | 拒絕連線（**生產環境建議**）|

#### action_generic_failure

發生某些失敗（如無法與本地 slurmd 通訊或核心不支援）時執行的動作。

| 值 | 說明 |
|----|------|
| **ignore**（預設）| 不做任何事，繼續到下一個 PAM 模組（**警告**：僅建議測試用）|
| **allow** | 允許連線但不納入（**警告**：僅建議測試用）|
| **deny** | 拒絕連線（**生產環境建議**）|

#### disable_x11

關閉 Slurm 內建的 X11 轉發支援。

| 值 | 說明 |
|----|------|
| **0**（預設）| 如果作業啟用了 Slurm X11 轉發，會覆寫 DISPLAY 變數 |
| **1** | 不檢查 Slurm X11 轉發支援，不修改 DISPLAY 變數 |

#### join_container

控制與命名空間外掛程式的互動。

| 值 | 說明 |
|----|------|
| **true**（預設）| 嘗試加入命名空間外掛程式建立的命名空間 |
| **false** | 不嘗試加入命名空間 |

#### log_level

參見 slurm.conf 中的 SlurmdDebug。預設為 **info**。

#### nodename

如果 slurm.conf 中定義的 NodeName 與此節點的主機名稱（由 `hostname -s` 報告）不同，必須設定為此主機在 slurm.conf 中的 NodeName。

#### service

此模組應執行的 pam 服務名稱。預設只對 sshd 執行。可指定其他服務名稱如 "login" 或 "*"。

---

### 防火牆、IP 位址等

slurmd 應該可從使用者可能啟動 ssh 的任何 IP 位址存取。判斷來源作業的 RPC 必須能夠到達該特定 IP 位址上的 slurmd 連接埠。

如果來源節點（如登入節點）沒有 slurmd，最好讓 RPC 被拒絕而非靜默丟棄，這樣可以更好地回應 RPC 發起者。

---

### SELinux

SELinux 可能與 pam_slurm_adopt 衝突，但通常可以並存。

**基本類型強制檔案：**
```
module pam_slurm_adopt 1.0;

require {
    type sshd_t;
    type var_spool_t;
    type unconfined_t;
    type initrc_var_run_t;
    class sock_file write;
    class dir { read search };
    class unix_stream_socket connectto;
}

#============= sshd_t ==============
allow sshd_t initrc_var_run_t:dir search;
allow sshd_t initrc_var_run_t:sock_file write;
allow sshd_t unconfined_t:unix_stream_socket connectto;
allow sshd_t var_spool_t:dir read;
allow sshd_t var_spool_t:sock_file write;
```

**使用 namespace/tmpfs 時：**
```
module pam_slurm_adopt 1.0;

require {
    type nsfs_t;
    type var_spool_t;
    type initrc_var_run_t;
    type unconfined_t;
    type sshd_t;
    class sock_file write;
    class dir { read search };
    class unix_stream_socket connectto;
    class fd use;
    class file read;
    class capability sys_admin;
}

#============= sshd_t ==============
allow sshd_t initrc_var_run_t:dir search;
allow sshd_t initrc_var_run_t:sock_file write;
allow sshd_t nsfs_t:file read;
allow sshd_t unconfined_t:fd use;
allow sshd_t unconfined_t:unix_stream_socket connectto;
allow sshd_t var_spool_t:dir read;
allow sshd_t var_spool_t:sock_file write;
allow sshd_t self:capability sys_admin;
```

---

### 限制

1. 某些 AuthenticationMethods 會導致 sshd 在登入流程中 fork 額外程序，這可能會混淆 PAM 模組並中斷 pam_slurm_adopt 的程序納入

2. 使用 Slurm 的 SELinux 支援時，透過 pam_slurm_adopt 啟動的會話不一定與關聯的作業在相同的上下文中

3. 使用 namespace/linux 且設定了使用者命名空間時，pam_limits 模組可能無法設定 memlock、sigpending、msgqueue、nice 或 rtprio

---

## 說明

### 程序納入流程

```
使用者 SSH 連線
        │
        ▼
pam_slurm_adopt 檢查
        │
        ├── 使用者有作業？ ──否──► 拒絕（action_no_jobs=deny）
        │
        ▼ 是
        │
        ├── 判斷來源作業
        │       │
        │       ├── 單一作業 ──► 納入該作業
        │       │
        │       └── 多個作業 ──► RPC 定位或選擇最新（action_unknown）
        │
        ▼
納入作業的 extern 步驟
        │
        ▼
連線成功，受作業限制
```

### 外部步驟 (extern step)

- 設定 `PrologFlags=contain` 時為每個作業建立
- 用於納入 SSH 連線和衍生程序
- 確保作業結束時所有程序都被清理
- 程序受作業的 cgroup 限制

---

## 實務範例

### 完整生產環境設定

**slurm.conf：**
```
PrologFlags=contain
TaskPlugin=task/cgroup
ProctrackType=proctrack/cgroup
LaunchParameters=ulimit_pam_adopt
```

**/etc/pam.d/sshd：**
```
# 認證
auth       required     pam_sepermit.so
auth       include      password-auth
auth       required     pam_env.so

# 帳戶
account    required     pam_nologin.so
account    include      password-auth
-account   required     pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny

# 會話
session    required     pam_selinux.so close
session    required     pam_limits.so
session    required     pam_selinux.so open
session    include      password-auth
```

**/etc/ssh/sshd_config：**
```
UsePAM yes
AuthenticationMethods publickey,password
# 不要使用 keyboard-interactive
```

### 允許管理員存取

**/etc/security/access.conf：**
```
+:(admins):ALL
-:ALL:ALL
```

**/etc/pam.d/sshd（帳戶部分）：**
```
account    sufficient    pam_access.so
account    required      pam_nologin.so
account    include       password-auth
-account   required      pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 PrologFlags=contain | 必須在使用模組前設定 |
| 使用 keyboard-interactive 認證 | 僅使用 publickey 或 password |
| 未停用 pam_systemd | 在所有相關 PAM 檔案中停用 |
| 測試設定用於生產環境 | 設定 action_*_failure=deny |
| pam_slurm_adopt 順序錯誤 | 應在 account 堆疊的最後 |

### 疑難排解

```bash
# 檢查 PAM 設定
cat /etc/pam.d/sshd

# 檢查 SSH 認證方法
grep AuthenticationMethods /etc/ssh/sshd_config

# 檢查 Slurm 設定
scontrol show config | grep -E "PrologFlags|TaskPlugin|ProctrackType"

# 檢查作業的 extern 步驟
scontrol show step <jobid>.extern

# 查看 PAM 日誌
journalctl -u sshd | grep pam
```

---

## 快速參考

### slurm.conf 設定

```
PrologFlags=contain
TaskPlugin=task/cgroup
ProctrackType=proctrack/cgroup
LaunchParameters=ulimit_pam_adopt
```

### PAM 設定

```
# 測試用
-account   required   pam_slurm_adopt.so

# 生產環境
-account   required   pam_slurm_adopt.so action_adopt_failure=deny action_generic_failure=deny
```

### 模組選項摘要

| 選項 | 預設值 | 建議生產值 |
|------|--------|------------|
| action_no_jobs | deny | deny |
| action_unknown | newest | newest 或 deny |
| action_adopt_failure | allow | deny |
| action_generic_failure | ignore | deny |
| disable_x11 | 0 | 根據需求 |
| join_container | true | true |
| log_level | info | info |

### 相關文件

- [cgroups 指南](cgroups.md) - cgroup 設定
- [Prolog/Epilog](prolog_epilog.md) - Prolog/Epilog 腳本
- [快速入門管理指南](quickstart_admin.md) - 基本設定
