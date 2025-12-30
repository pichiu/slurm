# Slurm NSS 外掛程式 (nss_slurm)

## TL;DR

nss_slurm 是選用的 NSS 外掛程式，允許在運算節點上透過本地 slurmstepd 程序處理作業的 passwd、group 和雲端節點主機解析，而非透過 LDAP、DNS 等網路服務。設定 `LaunchParameters=enable_nss_slurm`，在 `/etc/nsswitch.conf` 添加 slurm。可解決大規模作業啟動時的「驚群效應」問題。

---

## 翻譯

### 概觀

nss_slurm 是選用的 NSS 外掛程式，允許在運算節點上透過本地 slurmstepd 程序處理作業的 passwd、group 和雲端節點主機解析，而非透過 LDAP、DNS、SSSD 或 NSLCD 等網路服務。

#### passwd 資訊

當在叢集上啟用時，每個作業的使用者將有其完整的 **struct passwd** 資訊安全地作為每個步驟啟動的一部分傳送，並快取在 slurmstepd 程序中：
- 使用者名稱
- uid
- 主要 gid
- gecos 資訊
- 家目錄
- shell

此資訊將透過 `getpwuid()`/`getpwnam()`/`getpwent()` 系統呼叫提供給該步驟啟動的任何程序。

#### group 資訊

對於群組資訊（來自 `getgrgid()`/`getgrnam()`/`getgrent()` 系統呼叫），將提供 **struct group** 的簡略檢視。在給定程序中，回應將只包含使用者所屬的群組，但成員只列出使用者本身。不提供完整的群組成員清單。

#### host 資訊

對於主機資訊（來自 `gethostbyname()`/`gethostbyname` 系統呼叫），將提供 **struct hostent** 的簡略檢視。在給定程序中，回應將只包含屬於分配的雲端主機。

所有這些資訊由 slurmctld 填充，如同在執行 slurmctld 的主機上所見。

---

### 安裝

#### 從原始碼安裝

在 Slurm 建置目錄中，導航到 `contribs/nss_slurm/` 並執行：

```bash
make && make install
```

這將在安裝路徑中與其他 Slurm 函式庫檔案一起安裝 `libnss_slurm.so.2`。

#### 建立符號連結

根據您的 Linux 發行版，需要將其符號連結到包含其他 NSS 外掛程式的目錄：

| 發行版 | 建議路徑 |
|--------|----------|
| Debian/Ubuntu | `/lib/x86_64-linux-gnu` |
| RHEL 系列 | `/usr/lib64` |

如有疑問，執行：
```bash
find /lib /usr/ -name 'libnss*'
```

---

### 設定

#### Slurm 設定

slurmctld 必須設定為查詢並傳送適當的 passwd 和 group 詳細資訊作為啟動憑證的一部分：

```
# slurm.conf
LaunchParameters=enable_nss_slurm
```

設定後重新啟動 slurmctld。

#### 驗證設定

啟用後，可以在運算節點上使用 `scontrol getent` 命令列印與該節點上作業步驟相關的所有 passwd 和 group 資訊：

```bash
tim@node0001:~$ scontrol getent node0001
JobId=1268.Extern:
User:
tim:x:1000:1000:Tim Wickberg:/home/tim:/bin/bash
Groups:
tim:x:1000:tim
projecta:x:1001:tim

JobId=1268.0:
User:
tim:x:1000:1000:Tim Wickberg:/home/tim:/bin/bash
Groups:
tim:x:1000:tim
projecta:x:1001:tim
```

---

### nss_slurm 設定檔

nss_slurm 有選用的設定檔：`/etc/nss_slurm.conf`。只有在以下情況才需要此設定檔：

| 情況 | 設定選項 |
|------|----------|
| 節點主機名稱不符合 NodeName | 必須明確設定 `NodeName` |
| SlurmdSpoolDir 不符合預設 `/var/spool/slurmd` | 必須提供 `SlurmdSpoolDir` |

目前只支援 NodeName 和 SlurmdSpoolDir 設定選項。

---

### 初始測試

在節點上直接啟用 NSS Slurm 之前，應在新啟動的作業步驟中使用 `getent` 的 `-s slurm` 選項來驗證設定是否成功完成。

`-s` 選項允許 getent 查詢特定資料庫，即使它尚未透過系統的 `nsswitch.conf` 預設啟用。

**注意**：nss_slurm 只回應來自作業步驟內程序的請求，必須在作業步驟中啟動 getent 命令才能看到資料。

```bash
tim@blackhole:~$ srun getent -s slurm passwd
tim:x:1000:1000:Tim Wickberg:/home/tim:/bin/bash

tim@blackhole:~$ srun getent -s slurm group
tim:x:1000:tim
projecta:x:1001:tim
```

---

### NSS 設定

啟用 nss_slurm 只需在 `/etc/nsswitch.conf` 的 passwd 和 group 資料庫中添加 `slurm`（僅在執行 slurmd 的系統上）。

**建議**：將 `slurm` 列在第一位，因為順序（從左到右）決定 NSS 資料庫的查詢順序，這確保 Slurm 在能夠處理請求時先處理，然後再提交到其他來源。

**雲端節點名稱解析**：需要在 `/etc/nsswitch.conf` 的 hosts 資料庫中添加 `slurm`，建議將其列在最後。

```
# /etc/nsswitch.conf
passwd:     slurm files ldap
group:      slurm files ldap
hosts:      files dns slurm
```

啟用後測試：

```bash
tim@blackhole:~$ srun getent passwd tim
tim:x:1000:1000:Tim Wickberg:/home/tim:/bin/bash

tim@blackhole:~$ srun getent group projecta
projecta:x:1001:tim
```

---

### 限制

1. **僅限作業步驟內**：
   - nss_slurm 只為給定作業步驟內的程序返回結果
   - 不會為步驟外的程序返回結果，如系統監控、節點健康檢查、prolog 或 epilog 腳本等

2. **非完整替代**：
   - nss_slurm 不是 LDAP 等網路目錄服務的完整替代
   - 設計為減輕這些系統的負載以改善大規模作業啟動效能
   - 透過移除「驚群效應」（大型作業的所有任務同時查詢）來實現

3. **單一 slurmd**：
   - nss_slurm 只能與單一 slurmd 通訊
   - 如果使用 `--enable-multiple-slurmd`，可在 `nss_slurm.conf` 中指定 NodeName 和 SlurmdSpoolDir

4. **資訊差異問題**：
   - 資訊從 slurmctld 節點收集
   - 如果控制器和工作節點之間的資訊不同，可能有意外後果
   - 例如：如果使用者的 shell 在 slurmctld 機器上是 `/sbin/nologin`，但在 slurmd 節點上是 `/bin/bash`，互動式 salloc 可能無法啟動

5. **proctrack/pgid 限制**：
   - 使用 proctrack/pgid 時，nss_slurm 依賴程序的 pgid 決定是否回應請求
   - `srun --pty` 啟動的登入 shell 必須在自己的 session（因此自己的 pgid）中執行
   - nss_slurm 不會回應互動式 session 中的請求

---

## 說明

### nss_slurm 如何解決驚群效應

```
傳統方式（無 nss_slurm）：
                          ┌─→ LDAP/DNS
Task 0 ───→ getpwuid() ───┤
Task 1 ───→ getpwuid() ───┼─→ LDAP/DNS  ← 同時大量查詢
Task 2 ───→ getpwuid() ───┤              導致過載
  ...                     └─→ LDAP/DNS

使用 nss_slurm：
                          ┌─→ slurmstepd（本地快取）
Task 0 ───→ getpwuid() ───┤
Task 1 ───→ getpwuid() ───┼─→ slurmstepd（本地快取）← 無網路負載
Task 2 ───→ getpwuid() ───┤
  ...                     └─→ slurmstepd（本地快取）
```

---

## 實務範例

### 完整設定流程

```bash
# 1. 編譯並安裝
cd $SLURM_BUILD/contribs/nss_slurm/
make && make install

# 2. 建立符號連結（RHEL/CentOS）
ln -s /usr/lib64/slurm/libnss_slurm.so.2 /usr/lib64/libnss_slurm.so.2

# 3. 設定 slurm.conf
# LaunchParameters=enable_nss_slurm

# 4. 重新啟動 slurmctld
systemctl restart slurmctld

# 5. 設定 nsswitch.conf（在運算節點上）
# /etc/nsswitch.conf:
# passwd:     slurm files
# group:      slurm files
```

### nss_slurm.conf 範例

```
# /etc/nss_slurm.conf
# 如果主機名稱不符合 NodeName
NodeName=compute001

# 如果 SlurmdSpoolDir 非預設
SlurmdSpoolDir=/var/spool/slurmd
```

### 驗證

```bash
# 測試特定 slurm 資料庫
srun getent -s slurm passwd $USER
srun getent -s slurm group

# 測試完整 NSS 堆疊
srun getent passwd $USER
srun getent group $USER

# 查看節點上的快取資訊
scontrol getent $(hostname)
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未在 slurm.conf 啟用 | 設定 `LaunchParameters=enable_nss_slurm` |
| 符號連結位置錯誤 | 檢查發行版的正確路徑 |
| nsswitch.conf 順序錯誤 | passwd/group 中 slurm 應在前面 |
| 在步驟外測試 | 必須在 srun 內測試 |

### 建議

1. **逐步啟用**：
   - 先用 `-s slurm` 測試
   - 確認正常後再修改 nsswitch.conf

2. **監控效能**：
   - 比較啟用前後的作業啟動時間
   - 檢查 LDAP/DNS 負載變化

3. **保持備援**：
   ```
   # nsswitch.conf 保留其他來源
   passwd: slurm files ldap
   group:  slurm files ldap
   ```

---

## 快速參考

### slurm.conf 設定

```
LaunchParameters=enable_nss_slurm
```

### nsswitch.conf 設定

```
passwd:     slurm files [ldap]
group:      slurm files [ldap]
hosts:      files dns slurm
```

### nss_slurm.conf 選項

| 選項 | 說明 |
|------|------|
| NodeName | 節點名稱（如與主機名不同）|
| SlurmdSpoolDir | Slurmd spool 目錄 |

### 常用命令

| 命令 | 功能 |
|------|------|
| `srun getent -s slurm passwd` | 測試 passwd 查詢 |
| `srun getent -s slurm group` | 測試 group 查詢 |
| `scontrol getent <node>` | 查看節點快取資訊 |

### 相關文件

- [PAM 模組](pam_slurm_adopt.md) - PAM 整合
- [Prolog/Epilog](prolog_epilog.md) - 腳本執行
