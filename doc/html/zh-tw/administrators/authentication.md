# Slurm 認證外掛程式 (Authentication Plugins)

## TL;DR

Slurm 提供多種認證機制驗證 RPC 請求的合法性：MUNGE（預設，使用共用金鑰）、auth/slurm（23.11+ 內建認證）和 JWT（用於用戶端到伺服器通訊）。所有叢集主機必須共用相同的加密金鑰。auth/slurm 從 24.05 起支援多金鑰輪換。

---

## 翻譯

### 概觀

了解 Slurm 接收的遠端程序呼叫 (RPC) 來自可信任來源非常重要。Slurm 內有幾種不同的認證機制可用於驗證請求的合法性和完整性。

---

### MUNGE

MUNGE 可用於建立和驗證憑證。它允許 Slurm 認證來自另一台具有匹配使用者和群組的主機的請求的 UID 和 GID。建置 Slurm 時必須存在 MUNGE 函式庫才能使用 munge 進行認證。叢集中的所有主機必須有共用的加密金鑰。

#### 設定步驟

1. **建立金鑰檔案**

   MUNGE 要求在執行 slurmctld、slurmdbd 和節點的機器上有共用金鑰：
   ```bash
   dd if=/dev/random of=/etc/munge/munge.key bs=1024 count=1
   ```

2. **設定權限**

   此金鑰應由 "munge" 使用者擁有，且不應被其他使用者讀取或寫入：
   ```bash
   chown munge:munge /etc/munge/munge.key
   chmod 400 /etc/munge/munge.key
   ```

3. **分發金鑰檔案**

   將金鑰檔案分發到叢集中的機器。它需要在執行 slurmctld、slurmdbd、slurmd 和任何您設定的提交主機上。

4. **啟動 MUNGE 服務**
   ```bash
   systemctl enable munge
   systemctl start munge
   ```

5. **更新設定檔**

   變更認證機制需要重新啟動 Slurm 守護程式。在更新 slurm.conf 之前需要停止守護程式。

   **slurm.conf：**
   ```
   AuthType = auth/munge
   CredType = cred/munge
   ```

   **slurmdbd.conf：**
   ```
   AuthType = auth/munge
   ```

6. **重新啟動 Slurm 守護程式**

---

### Slurm 內建認證（auth/slurm）

從 23.11 版本開始，Slurm 有自己的外掛程式可以建立和驗證憑證。它驗證請求來自其他具有匹配使用者和群組的主機上的合法 UID 和 GID。叢集中的所有主機必須有共用的加密金鑰。

#### 單一金鑰設定

1. **建立金鑰檔案**
   ```bash
   dd if=/dev/random of=/etc/slurm/slurm.key bs=1024 count=1
   ```

2. **設定權限**

   slurm.key 或 slurm.jwks 應由 SlurmUser 擁有，且不應被其他使用者讀取或寫入：
   ```bash
   chown slurm:slurm /etc/slurm/slurm.key
   chmod 600 /etc/slurm/slurm.key
   ```

3. **分發金鑰檔案**

   將金鑰檔案分發到執行 slurmctld、slurmdbd、slurmd 和 sackd 的機器。

4. **更新設定檔**

   **slurm.conf：**
   ```
   AuthType = auth/slurm
   CredType = cred/slurm
   ```

   **slurmdbd.conf：**
   ```
   AuthType = auth/slurm
   ```

5. **建立執行時目錄**（如果不使用 systemd）
   ```bash
   mkdir /run/slurmctld /run/slurmdbd
   chown slurm:slurm /run/slurmctld /run/slurmdbd
   ```

6. **重新啟動守護程式**

#### 多金鑰設定（24.05+）

從 24.05 版本開始，您可以建立包含多個金鑰定義的 slurm.jwks 檔案。slurm.jwks 檔案有助於金鑰輪換，因為在輪換金鑰時不需要同時重新啟動叢集。相反，執行 `scontrol reconfigure` 即可。

**slurm.jwks 範例：**
```json
{
  "keys": [
    {
      "alg": "HS256",
      "kty": "oct",
      "kid": "key-identifier",
      "k": "VGhlIGtleSBiZWxvdyBtZSBuZXZlciBsaWVz",
      "exp": 1718200800
    },
    {
      "alg": "HS256",
      "kty": "oct",
      "kid": "key-identifier-2",
      "k": "VGhlIGtleSBhYm92ZSBtZSBhbHdheXMgbGllcw==",
      "use": "default"
    }
  ]
}
```

**欄位說明：**

| 欄位 | 說明 | 必要性 |
|------|------|--------|
| `alg` | 與金鑰一起使用的加密演算法，值必須為 HS256 | 必要 |
| `kty` | 用於簽署金鑰的加密演算法系列，值必須為 oct | 必要 |
| `kid` | 用於金鑰匹配的區分大小寫文字識別碼，必須唯一 | 必要 |
| `k` | 實際金鑰，以 Base64 或 Base64url 編碼的二進位資料表示，必須超過 16 位元組 | 必要 |
| `use` | 決定此金鑰是否為預設金鑰，僅接受 "default" 值，只能有一個預設金鑰 | 選用 |
| `exp` | 金鑰的到期日期，以 Unix 時間戳記表示 | 選用 |

#### SACK（Slurm Auth and Cred Kiosk）

Slurm 的內部認證使用一個子系統 — **S**lurm **A**uth and **C**red **K**iosk (SACK) — 負責處理來自 **auth/slurm** 和 **cred/slurm** 外掛程式的請求。此子系統由 slurmctld、slurmdbd 和 slurmd 守護程式在每個系統上自動啟動和內部管理，無需執行單獨的守護程式。

對於未執行這些 Slurm 守護程式的登入節點，應執行 **sackd** 守護程式以允許 Slurm 用戶端命令向叢集其餘部分進行認證。此守護程式也可以管理[無設定檔](configless_slurm.md)環境的快取設定檔。

從 25.05 版本開始，可以在同一登入節點上共存多個 **sackd** 守護程式，方法是在不同的 systemd 服務檔案中變更 RuntimeDirectory 選項。

---

### JWT

Slurm 可以設定為使用 JSON Web Tokens (JWT) 進行認證。這透過 AuthAltType 參數設定，僅用於用戶端到伺服器的通訊。更多資訊請參閱 [JWT 認證指南](jwt.md)。

---

## 說明

### 認證類型比較

| 認證類型 | 版本 | 適用場景 | 優點 | 缺點 |
|----------|------|----------|------|------|
| auth/munge | 所有版本 | 傳統叢集 | 成熟穩定、廣泛使用 | 需要額外的 MUNGE 守護程式 |
| auth/slurm | 23.11+ | 新叢集 | 無需外部相依、支援多金鑰 | 較新，功能仍在發展 |
| JWT | 所有版本 | REST API、slurmrestd | 適合程式化存取 | 僅用於用戶端到伺服器 |

### 金鑰分發方式

```
控制節點 (slurmctld)
     ↓
  共用金鑰檔案
     ↓
├── DBD 節點 (slurmdbd)
├── 計算節點 (slurmd)
└── 登入節點 (sackd)
```

---

## 實務範例

### 從 MUNGE 遷移到 auth/slurm

```bash
# 1. 建立新金鑰
dd if=/dev/random of=/etc/slurm/slurm.key bs=1024 count=1
chown slurm:slurm /etc/slurm/slurm.key
chmod 600 /etc/slurm/slurm.key

# 2. 分發金鑰到所有節點
pdcp -a /etc/slurm/slurm.key /etc/slurm/

# 3. 停止所有 Slurm 守護程式
systemctl stop slurmctld slurmdbd
pdsh -a "systemctl stop slurmd"

# 4. 更新設定
# slurm.conf:
#   AuthType = auth/slurm
#   CredType = cred/slurm
# slurmdbd.conf:
#   AuthType = auth/slurm

# 5. 重新啟動守護程式
systemctl start slurmdbd
systemctl start slurmctld
pdsh -a "systemctl start slurmd"
```

### 設定多金鑰輪換

```bash
# 1. 產生 Base64 編碼金鑰
NEW_KEY=$(dd if=/dev/random bs=32 count=1 2>/dev/null | base64)

# 2. 建立 slurm.jwks 檔案
cat > /etc/slurm/slurm.jwks << EOF
{
  "keys": [
    {
      "alg": "HS256",
      "kty": "oct",
      "kid": "key-2024-01",
      "k": "${NEW_KEY}",
      "use": "default"
    }
  ]
}
EOF

# 3. 設定權限
chown slurm:slurm /etc/slurm/slurm.jwks
chmod 600 /etc/slurm/slurm.jwks

# 4. 分發到所有節點並重新設定
pdcp -a /etc/slurm/slurm.jwks /etc/slurm/
scontrol reconfigure
```

### 驗證認證設定

```bash
# 檢查目前認證類型
scontrol show config | grep -i auth

# 測試認證
srun -N1 hostname

# 檢查 MUNGE 狀態
munge -n | unmunge

# 檢查 sackd 狀態（auth/slurm）
systemctl status sackd
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 金鑰權限過寬 | MUNGE: chmod 400；auth/slurm: chmod 600 |
| 金鑰未分發到所有節點 | 確保所有節點都有相同的金鑰檔案 |
| 忘記重新啟動守護程式 | 變更認證類型後必須重新啟動所有守護程式 |
| 時鐘不同步 | MUNGE 要求叢集時鐘同步（使用 NTP） |
| 混用認證類型 | 所有節點必須使用相同的認證類型 |

### 除錯技巧

```bash
# 測試 MUNGE 認證
munge -n | ssh node01 unmunge

# 檢查金鑰檔案權限
ls -la /etc/munge/munge.key
ls -la /etc/slurm/slurm.key

# 查看認證相關日誌
journalctl -u munge
journalctl -u slurmctld | grep -i auth

# 驗證所有節點金鑰一致
pdsh -a "md5sum /etc/slurm/slurm.key"
```

---

## 快速參考

### 認證設定摘要

**MUNGE：**
```
# slurm.conf
AuthType = auth/munge
CredType = cred/munge

# 金鑰位置
/etc/munge/munge.key
```

**auth/slurm：**
```
# slurm.conf
AuthType = auth/slurm
CredType = cred/slurm

# 金鑰位置
/etc/slurm/slurm.key    # 單一金鑰
/etc/slurm/slurm.jwks   # 多金鑰（24.05+）
```

### 金鑰權限

| 認證類型 | 擁有者 | 權限 |
|----------|--------|------|
| MUNGE | munge:munge | 400 |
| auth/slurm | slurm:slurm | 600 |

### 相關守護程式

| 守護程式 | 功能 |
|----------|------|
| munged | MUNGE 認證守護程式 |
| sackd | auth/slurm 登入節點守護程式 |
| slurmctld | 控制器（內建 SACK） |
| slurmdbd | 資料庫（內建 SACK） |
| slurmd | 計算節點（內建 SACK） |

### 相關文件

- [JWT 認證](jwt.md)
- [無設定檔操作](configless_slurm.md)
- [管理員快速入門](quickstart_admin.md)
