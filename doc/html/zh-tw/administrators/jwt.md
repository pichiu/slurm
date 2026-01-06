# Slurm JSON Web Tokens (JWT) 認證

## TL;DR

Slurm 提供符合 RFC7519 的 JSON Web Tokens (JWT) 實作。JWT 可作為 `AuthAltType` 使用，通常搭配 `auth/munge` 作為 `AuthType`。設定 `AuthAltTypes=auth/jwt` 並提供金鑰檔案，使用 `scontrol token` 產生權杖。也支援 JWKS 與 RS256 權杖整合外部認證系統（如 Keycloak、Amazon Cognito、Azure AD）。

---

## 翻譯

### 概觀

Slurm 提供符合 [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519) 的 [JSON Web Tokens (JWT)](https://jwt.io/) 實作。此認證可作為 **AuthAltType** 使用，通常搭配 **auth/munge** 作為 **AuthType**。

**支援的通訊方向**：僅支援從客戶端連接到 **slurmctld** 和 **slurmdbd**。

**限制**：某些情境（特別是使用 **srun** 的互動式作業）目前不支援啟用 auth/jwt 的客戶端（或環境變數中有 SLURM_JWT 的客戶端）。

---

### 先決條件

JWT 需要 [libjwt](related_software.html#jwt)。編譯 Slurm 時，函式庫和開發標頭都必須可用。

---

### JWT 建立的完全 root 信任

建立 JWT 的能力等同於叢集上的 root 權限。這是每個站點關於信任什麼/誰/何時/如何的決定。

如果給定的認證系統無法被完全信任擁有整個叢集的 root 權限，則需要使用認證代理來分割信任並在請求到達 Slurm（特別是 slurmrestd）之前實施站點的特定政策。

一旦作業進入佇列，代理認證系統將不再參與，作業將以該使用者的權限和存取權執行，遵循 Linux/POSIX 的 ACL 和信任。

---

### 信任模型

有幾種方式可以處理控制 JWT 認證和存取。Slurm JWT 外掛程式實作故意保持簡單，無法支援站點所需的大多數信任模型。

| 模型 | 說明 |
|------|------|
| **外部 JWT 產生** | 使用外部工具產生 JWT，最簡單但需要站點實作工具 |
| **認證代理** | 最靈活，任何認證系統都可放在 slurmrestd 前面 |
| **JWKS** | 跳過認證系統在 Slurm 前面，使用簽署的公開金鑰 |

#### 認證代理

認證代理是最靈活的選項。它需要建立 slurmuser/root 權杖，然後可用來代理任何使用者。

**必要標頭**：
- `X-SLURM-USER-TOKEN`
- `X-SLURM-USER-NAME`

認證代理不需要為客戶端實作 JWT。這是認證代理的主要優勢 — 它們可以使用任何認證方法，因為它們是告訴 Slurm 請求來自哪個使用者的信任點。

---

### 獨立使用設定

#### 步驟 1：建構 Slurm

[建構支援 JWT 的 Slurm](related_software.html#jwt)

#### 步驟 2：產生金鑰

將相同的 JWT 金鑰新增到控制器和 slurmdbd（如果使用）。建議將 JWT 金鑰放在 StateSaveLocation。

```bash
dd if=/dev/random of=/var/spool/slurm/statesave/jwt_hs256.key bs=32 count=1
chown slurm:slurm /var/spool/slurm/statesave/jwt_hs256.key
chmod 0600 /var/spool/slurm/statesave/jwt_hs256.key
chown slurm:slurm /var/spool/slurm/statesave
chmod 0755 /var/spool/slurm/statesave
```

**注意**：
- 金鑰不必在 StateSaveLocation，但如果有多個控制器，這是方便的位置
- 金鑰不應放在非管理員使用者可能存取的目錄
- 金鑰檔案應由 **SlurmUser** 或 **root** 擁有，建議權限 0400
- 檔案不能被 'other' 存取

#### 步驟 3：設定

在 slurm.conf 和 slurmdbd.conf 中新增 JWT 作為替代認證類型：

```
AuthAltTypes=auth/jwt
AuthAltParameters=jwt_key=/var/spool/slurm/statesave/jwt_hs256.key
```

#### 步驟 4：重新啟動 slurmctld

#### 步驟 5：建立權杖

```bash
# 為特定使用者建立權杖（root 或 SlurmUser）
scontrol token username=$USER

# 使用者為自己建立權杖
scontrol token

# 指定有效期限（預設 1800 秒）
scontrol token lifespan=3600
```

**禁止使用者產生權杖**：
```
AuthAltParameters=disable_token_creation
```

#### 步驟 6：使用權杖

```bash
# 設定環境變數
export SLURM_JWT=<token>

# 啟動 slurmrestd 時使用 JWT 作為主要認證
export SLURM_JWT=daemon
slurmrestd ...
```

---

### 外部認證整合（JWKS 和 RS256 權杖）

從 21.08 版本開始，Slurm 可支援 RS256 權杖，例如由以下服務產生的權杖：

- [Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html)
- [Azure AD](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters/oidc-issuer/jwks.html)
- [Keycloak](https://www.keycloak.org/docs/latest/securing_apps/#_client_authentication_adapter)

#### 啟用 RS256 權杖支援

下載並設定 JWKS 檔案：

```
AuthAltTypes=auth/jwt
AuthAltParameters=jwks=/var/spool/slurm/statesave/jwks.json
```

**注意**：
- jwks 檔案應由 **SlurmUser** 或 **root** 擁有
- 必須可被 **SlurmUser** 讀取，建議權限 0400
- 檔案不能被 'other' 寫入
- 啟用 JWKS 支援時，內建 HS256 權杖產生功能會被停用
- 可透過同時設定 `jwt_key=` 和 `jwks=` 重新啟用

**重要**：Slurm 忽略 **x5c** 和 **x5t** 欄位，不會驗證 JWKS 檔案中的憑證鏈。JWT 僅透過 **e** 和 **n** 欄位提供的 RSA 256 位元金鑰驗證。

---

### Keycloak 整合

#### 基本設定

**startup.sh**：
```bash
#!/bin/bash
export SECRET=secret
/opt/keycloak/bin/kc.sh bootstrap-admin service --client-id test --client-secret:env=SECRET
exec /opt/keycloak/bin/kc.sh start-dev --log=console --log-console-level=debug --http-enabled true
```

**Dockerfile**：
```dockerfile
FROM keycloak/keycloak
COPY startup.sh /startup.sh
ENTRYPOINT [ "/startup.sh" ]
```

**環境變數**：
```
KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=password
```

#### 使用 JWKS

1. 建立包含有效 RS256 金鑰的 JSON 檔案：
```bash
curl "http://keycloak:8080/realms/master/protocol/openid-connect/certs" > /etc/slurm/jwks.json
```

2. 在 Slurm 中指向該檔案：
```
AuthAltTypes=auth/jwt
AuthAltParameters=jwks=/etc/slurm/jwks.json,userclaimfield=preferred_username
```

#### 使用 JWT

1. 建立從 KeyCloak 產生 JWT 的腳本：
```bash
#!/bin/bash
[ -z "$1" -o -z "$2" ] && echo "USAGE: $0 {user_name} {user_password}" && exit 1

curl -s \
  -d "client_id=test" \
  -d "client_secret=secret" \
  -d "username=$1" \
  -d "password=$2" \
  -d "grant_type=password" \
  -d "scope=openid" \
  "http://keycloak:8080/realms/master/protocol/openid-connect/token" | \
  jq -r '.id_token'
```

2. 使用腳本設定 JWT：
```bash
env SLURM_JWT=$(get_keycloak_jwt.sh username password) \
sbatch -o none -e none --wrap 'unset SLURM_JWT; srun uptime'
```

---

### 使用者對應

Slurm 預設使用 `sun`（Slurm UserName）欄位。如果服務使用不同欄位，需要更正。

#### 選項 1：更改 Slurm 使用的欄位

```
AuthAltParameters=jwks=/local/path/to/jwks.json,userclaimfield=preferred_username
```

#### 選項 2：更改身分服務使用的欄位

在 KeyCloak 中：**Clients → Client details → Dedicated scopes → Mapper details**，更改使用者名稱對應使用 `sun` 欄位。

---

### 相容性

Slurm 使用 libjwt 檢視和驗證 [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519) JWT 權杖。

**JWT 必要欄位**：
- **iat** - 建立日期的 Unix 時間戳
- **exp** - 到期日期的 Unix 時間戳
- **sun 或 username** - Slurm 使用者名稱（POSIX.1-2017 使用者名稱）

**支援的演算法**：
- HS256 - 用於產生和驗證權杖
- RS256 - 僅用於驗證權杖（Slurm 無法直接建立）

---

## 說明

### JWT vs JWKS

| 特性 | JWT (HS256) | JWKS (RS256) |
|------|-------------|--------------|
| 金鑰類型 | 對稱金鑰 | 非對稱金鑰對 |
| 產生權杖 | Slurm 可直接產生 | 需要外部系統 |
| 適用場景 | 簡單部署 | 整合外部認證系統 |
| 金鑰管理 | 單一金鑰 | JWKS 檔案 |

### 認證流程

```
使用者 → JWT 權杖 → slurmctld/slurmdbd
                      │
                      ▼
                  驗證權杖
                      │
                      ▼
                  提取使用者名稱
                      │
                      ▼
                  授權操作
```

---

## 實務範例

### 基本 JWT 設定

**slurm.conf：**
```
AuthType=auth/munge
AuthAltTypes=auth/jwt
AuthAltParameters=jwt_key=/var/spool/slurm/statesave/jwt_hs256.key
```

**slurmdbd.conf：**
```
AuthType=auth/munge
AuthAltTypes=auth/jwt
AuthAltParameters=jwt_key=/var/spool/slurm/statesave/jwt_hs256.key
```

### Python 產生 JWT 腳本

```python
#!/usr/bin/env python3
import sys
import time
from jwt import JWT
from jwt.jwa import HS256
from jwt.jwk import jwk_from_dict
from jwt.utils import b64encode

if len(sys.argv) != 3:
    sys.exit("gen_jwt.py [user name] [expiration time (seconds)]")

with open("/var/spool/slurm/statesave/jwt.key", "rb") as f:
    priv_key = f.read()

signing_key = jwk_from_dict({
    'kty': 'oct',
    'k': b64encode(priv_key)
})

message = {
    "exp": int(time.time() + int(sys.argv[2])),
    "iat": int(time.time()),
    "sun": sys.argv[1]
}

a = JWT()
compact_jws = a.encode(message, signing_key, alg='HS256')
print("SLURM_JWT={}".format(compact_jws))
```

### Python 驗證 JWT 腳本

```python
#!/usr/bin/env python3
import sys
from jwt import JWT
from jwt.jwk import jwk_from_dict
from jwt.utils import b64encode

if len(sys.argv) != 2:
    sys.exit("verify_jwt.py [JWT Token]")

with open("/var/spool/slurm/statesave/jwt.key", "rb") as f:
    priv_key = f.read()

signing_key = jwk_from_dict({
    'kty': 'oct',
    'k': b64encode(priv_key)
})

a = JWT()
b = a.decode(sys.argv[1], signing_key, algorithms=["HS256"])
print(b)
```

### 搭配 REST API 使用

```bash
# 產生權杖
export SLURM_JWT=$(scontrol token lifespan=3600 | cut -d= -f2)

# 使用 curl 呼叫 REST API
curl -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
     -H "X-SLURM-USER-NAME: $USER" \
     http://slurmrestd:6820/slurm/v0.0.40/jobs
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 金鑰檔案權限過寬 | 設定權限為 0400 或 0600 |
| 金鑰檔案可被 other 存取 | 確保僅 SlurmUser/root 可存取 |
| 未安裝 libjwt | 編譯前安裝 libjwt 和開發標頭 |
| JWT 過期 | 增加 lifespan 或重新產生權杖 |
| 使用者名稱欄位不符 | 設定 userclaimfield 參數 |

### 安全建議

1. **保護金鑰檔案**：
   - 權限設為 0400
   - 僅 SlurmUser/root 可存取
   - 不要放在使用者可存取的目錄

2. **權杖有效期限**：
   - 使用合理的有效期限
   - 考慮站點安全需求

3. **禁用使用者產生權杖**：
   ```
   AuthAltParameters=disable_token_creation
   ```

4. **使用認證代理**：
   - 對於不完全信任的情況
   - 實施站點特定政策

---

## 快速參考

### slurm.conf 設定

```
# 基本 JWT 設定
AuthAltTypes=auth/jwt
AuthAltParameters=jwt_key=/path/to/jwt.key

# JWKS 設定
AuthAltTypes=auth/jwt
AuthAltParameters=jwks=/path/to/jwks.json

# 同時使用
AuthAltParameters=jwt_key=/path/to/jwt.key,jwks=/path/to/jwks.json

# 自訂使用者欄位
AuthAltParameters=jwks=/path/to/jwks.json,userclaimfield=preferred_username

# 禁用使用者產生權杖
AuthAltParameters=jwt_key=/path/to/jwt.key,disable_token_creation
```

### 產生金鑰

```bash
# 產生 32 位元組隨機金鑰
dd if=/dev/random of=/path/to/jwt.key bs=32 count=1
chmod 0400 /path/to/jwt.key
chown slurm:slurm /path/to/jwt.key
```

### scontrol 命令

| 命令 | 功能 |
|------|------|
| `scontrol token` | 為自己產生權杖 |
| `scontrol token username=X` | 為指定使用者產生權杖 |
| `scontrol token lifespan=N` | 指定有效期限（秒）|

### 環境變數

| 變數 | 說明 |
|------|------|
| `SLURM_JWT` | JWT 權杖 |
| `SLURM_JWT=daemon` | 啟動 slurmrestd 時使用 |

### 相關文件

- [認證外掛程式](authentication.md) - 認證概觀
- [REST API](rest_quickstart.md) - REST API 使用
- [slurm.conf](slurm.conf.html) - 主要設定檔
- [libjwt 安裝](related_software.html#jwt) - 函式庫安裝
