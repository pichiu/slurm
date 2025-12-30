# Slurm REST API 詳細說明

## TL;DR

Slurm 透過 slurmrestd 守護程式提供 REST API，使用 JWT 進行認證。slurmrestd 是無狀態的，每個請求獨立處理。支援 Inet 服務模式和監聽模式。建議在 slurmrestd 和客戶端之間設定快取代理。安全性方面，slurmrestd 不設計為直接面向網際網路，應使用 TLS 代理包裝。

---

## 翻譯

### 概觀

Slurm 透過 slurmrestd 守護程式提供 [REST API](https://restfulapi.net/)，使用 [JSON Web Tokens](jwt.md) 進行認證。此守護程式設計為允許客戶端透過 REST API 與 Slurm 通訊（除了命令列介面 (CLI) 或 C API）。

**另請參閱**：
- [REST API 快速入門指南](rest_quickstart.md)
- [REST API 方法和模型](rest_api.md)
- [slurmrestd man page](slurmrestd.html)
- [OpenAPI 外掛程式發行說明](openapi_release_notes.md)
- [REST API 客戶端指南](rest_clients.md)

---

### 無狀態

slurmrestd 是無狀態的，因為它不會在請求之間快取或儲存任何狀態。每個請求都在一個執行緒中處理，然後所有狀態都會被丟棄。對 slurmrestd 的任何請求都與 Slurm 控制器（slurmctld 或 slurmdbd）完全同步，只有在 HTTP 回應碼發送給客戶端後才被視為完成。slurmrestd 會在處理請求時保持客戶端連線開放。Slurm 資料庫命令在每個請求結束時，在請求中所有 API 呼叫成功後提交。

**建議**：強烈建議站點在 slurmrestd 和客戶端之間設定快取代理，以避免客戶端重複呼叫查詢，導致控制器上的使用量高於需要（並導致鎖競爭）。

---

### 運行模式

slurmrestd 目前支援兩種運行模式：Inet 服務模式和監聽模式。

#### Inet 服務模式

slurmrestd 守護程式作為 [Inet 服務](https://en.wikipedia.org/wiki/Inetd)，將 STDIN 和 STDOUT 視為客戶端。此模式允許客戶端使用 inetd、xinetd 或 systemd socket 啟用服務，避免需要在主機上始終運行守護程式。此模式為每個客戶端建立一個實例，不支援為不同客戶端重複使用同一個實例。

#### 監聽模式

slurmrestd 守護程式作為完整的 UNIX 服務，持續監聽新的 TCP 連線。每個連線和請求都獨立認證。

---

### 設定

slurmrestd 可透過環境變數或命令列參數設定。詳細資訊請參閱 **doc/man/man1/slurmrestd.8** man page 和 [REST API 快速入門指南](rest_quickstart.md)。

---

### 外掛程式

從 Slurm 20.11 開始，REST API 使用外掛程式進行認證和產生內容。從 Slurm 21.08 開始，OpenAPI 外掛程式可在 slurmrestd 守護程式外部使用，其他 slurm 命令可能提供或接受最新版本的 OpenAPI 格式輸出。此功能按命令提供。請參閱 [Data Parser 生命週期](rest_clients.md) 文件了解版本化端點的計劃生命週期。這些外掛程式可選擇性地透過命令列參數列出或選擇，如 [slurmrestd](slurmrestd.html) 文件所述。

---

### 高可用性

slurmrestd 對於其在高可用叢集中的部署是不可知的。守護程式可以在多個節點上運行，但不提供與其他實例的任何協調以進行負載平衡或故障轉移。如果需要此類功能，可以部署單獨的負載平衡器。負載平衡器應能夠將任何所需的認證資訊轉發到 slurmrestd 機器。

slurmrestd 系統允許的連線數也應受到限制，以免 slurmctld 被請求淹沒。注意 **slurmrestd** 的 `-t <THREAD COUNT>` 和 `--max-connections <count>` 選項、部署的節點數量以及運行 **slurmctld** 的機器規格。

---

### 安全性

Slurm REST API 的設計是為客戶端提供使用 REST 命令控制 Slurm 的必要功能。它**不是**設計為直接面向網際網路。僅支援未加密和未壓縮的 HTTP 通訊。slurmrestd 也沒有針對中間人攻擊或重放攻擊的保護。slurmrestd 應該只放置在與受信任客戶端通訊的受信任網路中。

**對於希望將 Slurm REST API 暴露到網際網路或叢集外部的站點**：
- 至少使用代理以 TLS v1.3（或更高版本）包裝所有通訊
- 新增監控以拒絕在網路周邊防火牆或 TLS 代理處重複嘗試無效登入的任何客戶端
- 建議透過代理進行任何客戶端過濾，以避免常見的網際網路爬蟲與 slurmrestd 通訊並浪費系統資源或甚至導致有效客戶端的更高延遲
- 建議站點為客戶端使用較短期限的 JWT token 並經常更新，可能透過非 Slurm JWT 產生器以避免必須強制執行 JWT 期限限制
- 建議站點使用認證代理來處理所有客戶端對站點偏好的單一登入 (SSO) 提供者的認證，而不是 Slurm **scontrol** 產生的 token。這將防止任何未認證的客戶端連接到 slurmrestd

Slurm REST API 是一個 HTTP 伺服器，應套用任何 Web 伺服器安全的所有一般可能預防措施。由於這些預防措施是站點特定的，強烈建議您與站點的安全小組合作，確保在連接到 slurmrestd 之前在代理處強制執行所有策略。

Slurm 盡量不給潛在攻擊者任何認證失敗的提示。這導致客戶端收到這個相當簡短的訊息：`Authentication failure`。當發生這種情況時，請查看相關 Slurm 守護程式（即 **slurmdbd**、**slurmctld** 或 **slurmd**）的日誌以了解實際問題的資訊。

---

#### JSON Web Token (JWT) 認證

slurmrestd 支援使用 [JWT 認證使用者](jwt.md)。JWT 可用於透過 REST 協定認證使用者。

| 標頭 | 說明 |
|------|------|
| X-SLURM-USER-NAME | 使用者名稱標頭 |
| X-SLURM-USER-TOKEN | JWT 標頭 |

SlurmUser 或 root 可以提供替代使用者名稱以作為給定使用者的代理。使用 JWT 認證時，slurmrestd 應作為唯一的**無特權**使用者和群組運行。slurmrestd 應在啟動時提供一個無效的 SLURM_JWT 環境變數以啟用 JWT 認證。這將允許使用者在向代理認證時提供自己的 JWT token，並確保防止任何可能的意外授權。

使用 JWT 時，重要的是在 slurm.conf 和 slurmdbd.conf 中為 slurmrestd 設定 **AuthAltTypes=auth/jwt**。

---

#### 本地認證

slurmrestd 支援使用 UNIX domain socket 讓核心認證本地使用者。預設情況下，slurmrestd 不會以 root 或 SlurmUser 啟動，或如果使用者的主要群組屬於 root 或 SlurmUser。slurmrestd 必須位於 Munge 安全域中才能在本地認證模式下運作並與 Slurm 通訊。

---

#### 認證代理

如果使用 JWT 認證不符合您的需求，有多種認證系統可供站點選擇。認證代理設定使用分配給 SlurmUser 的 JWT token，然後可用於代理叢集上的任何使用者。此能力僅允許 SlurmUser 和 root 使用者，所有其他 token 只能與其本地分配的使用者一起使用。

如果使用第三方認證代理，預期它將提供正確的 HTTP 標頭（**X-SLURM-USER-NAME** 和 **X-SLURM-USER-TOKEN**）給 slurmrestd 以及使用者的請求。

Slurm 對認證代理沒有任何要求，除了它符合 HTTP 1.1 且提供正確的 HTTP 標頭以允許客戶端認證。Slurm 將明確信任提供的 HTTP 標頭，無法驗證它們（除了代理的受信任 token **X-SLURM-USER-TOKEN**）。任何認證代理都需要遵循您站點的安全策略，並確保代理請求來自正確的使用者。這些要求是任何認證代理的標準要求，不是 Slurm 特定的。

---

### Python 指南

OpenAPI 工具可用於產生 Python 客戶端與 REST API 互動。以下範例是 API 版本 0.0.43 的，其他版本可能有一些差異。

#### 設定

1. 安裝 [openapi-generator-cli](https://openapi-generator.tech/docs/installation/)

2. 編譯客戶端函式庫：
```bash
slurmrestd --generate-openapi-spec > openapi.json
openapi-generator-cli generate -i openapi.json -g python -o py_api_client
```

3. （可選但建議）初始化並啟用 Python 虛擬環境

4. 安裝所需套件：
```bash
cd py_api_client/
pip install -r requirements.txt
```

5. 設定 Python 腳本（假設已設定有效的 'SLURM_JWT' 環境變數）：
```python
import os
import time
from openapi_client import SlurmApi
from openapi_client import SlurmdbApi
from openapi_client import ApiClient as Client
from openapi_client import Configuration as Config

c = Config()
c.host = "http://localhost:8080/"
c.access_token = os.getenv("SLURM_JWT")
if not c.access_token:
    raise KeyError("No SLURM_JWT set")
slurm = SlurmApi(Client(c))
slurmdb = SlurmdbApi(Client(c))

# srun 二進位檔案位置 + 其他相關二進位檔案
environment = ['PATH=/bin/:/sbin/:/home/slurm/bin/:/home/slurm/sbin/']
curr_dir = '/tmp'
```

---

#### 使用概觀

設定完成後，您可以使用 `openapi_client` 模組存取對應 REST API 中模型和方法的類別和函式。注意 REST API 和 Python 客戶端之間的命名慣例轉換：

| API 模型/方法 | 對應 Python |
|--------------|-------------|
| `v0.0.43_job_desc_msg` | `V0043JobDescMsg` 類別 |
| `POST /slurm/v0.0.43/job/submit` | `slurm_v0043_post_job_submit()` 函式 |

如果遇到任何錯誤，請查看 [REST 快速入門](rest_quickstart.md) 頁面上的常見問題。

---

#### 作業提交

此範例顯示如何填充作業提交請求和作業描述訊息以及所需的提交參數，以及如何發送 POST 請求提交作業。

```python
from openapi_client import V0043JobSubmitReq
from openapi_client import V0043JobDescMsg

# 填充作業提交請求和作業描述訊息
my_job = V0043JobSubmitReq(
    script='#!/bin/bash\nsrun sleep 300',
    job=V0043JobDescMsg(
        name='rest_test',
        partition='gpu',
        tres_per_job='gres:gpu:amd:4',
        time_limit={"set": True, "number": 5},
        required_nodes=["n2", "n4"],
        tasks=5,
        environment=environment,
        current_working_directory=curr_dir
    )
)

# 發送 POST 請求提交作業
submit_response = slurm.slurm_v0043_post_job_submit(my_job)
```

---

#### 作業、節點和預約控制

作業、節點和預約可以透過 Python 客戶端以類似方式管理。每個實體需要自己的 import，每個都有類似的查看、修改和刪除函式。GET 函式和一些 POST/DELETE 函式也可以使用**複數**形式（例如 `slurm_v0043_get_jobs()`）來影響多個實體。

**作業控制**：
| 操作 | 函式 |
|------|------|
| 查看 | `slurm_v0043_get_job()` |
| 新增（提交）| `slurm_v0043_post_job_submit()` |
| 修改 | `slurm_v0043_post_job()` |
| 刪除（取消）| `slurm_v0043_delete_job()` |

**節點控制**：
| 操作 | 函式 |
|------|------|
| 查看 | `slurm_v0043_get_node()` |
| 修改 | `slurm_v0043_post_node()` |
| 刪除 | `slurm_v0043_delete_node()` |

**預約控制**：
| 操作 | 函式 |
|------|------|
| 查看 | `slurm_v0043_get_reservation()` |
| 新增（建立）| `slurm_v0043_post_reservation()` |
| 修改 | `slurm_v0043_post_reservation()` |
| 刪除 | `slurm_v0043_delete_reservation()` |

**預約範例**：
```python
from openapi_client import V0043ReservationDescMsg

# GET 請求查詢預約
resp = slurm.slurm_v0043_get_reservations()

# 檢查 GET 請求輸出
if "important_jobs" in [resv.name for resv in resp.reservations]:
    resp = slurm.slurm_v0043_delete_reservation("important_jobs")

# POST 請求建立預約
slurm.slurm_v0043_post_reservation(
    V0043ReservationDescMsg(
        name="important_jobs",
        duration={"set": True, "number": 15},
        node_list=["n4", "n5"],
        start_time={"set": True, "number": int(time.time())},
        users=["slurm"],
        flags=["IGNORE_JOBS", "MAGNETIC", "DAILY"],
    )
)

# POST 請求修改預約
slurm.slurm_v0043_post_reservation(
    V0043ReservationDescMsg(
        name="important_jobs",
        duration={"set": True, "number": 20},
    )
)
```

---

#### 系統管理

可以使用 `slurm.slurm_v0043_get_reconfigure()` 函式啟動系統重新設定。系統資訊也可以使用以下 API 函式查看：

- `slurm.slurm_v0043_get_partitions()`
- `slurm.slurm_v0043_get_diag()`
- `slurm.slurm_v0043_get_licenses()`

**查看分割區資訊範例**：
```python
# GET 請求查詢分割區
resp = slurm.slurm_v0043_get_partitions()

# 檢查請求輸出以過濾特定分割區 QOS
qos_parts = [part for part in resp.partitions if 'sample' == part.qos.assigned]

# GET 請求查詢特定名稱的分割區
defq = slurm.slurm_v0043_get_partition("defq")

# 檢查請求輸出以獲取分割區上的節點
configured_nodes = defq.partitions[0].nodes.configured
```

---

## 說明

### slurmrestd 架構

```
                    ┌─────────────────────────────────────────────┐
                    │              負載平衡器/代理                 │
                    │           (TLS 終止、快取)                   │
                    └─────────────────┬───────────────────────────┘
                                      │
           ┌──────────────────────────┼──────────────────────────┐
           │                          │                          │
           ▼                          ▼                          ▼
    ┌─────────────┐            ┌─────────────┐            ┌─────────────┐
    │ slurmrestd  │            │ slurmrestd  │            │ slurmrestd  │
    │   節點 1    │            │   節點 2    │            │   節點 N    │
    └──────┬──────┘            └──────┬──────┘            └──────┬──────┘
           │                          │                          │
           └──────────────────────────┼──────────────────────────┘
                                      │
                    ┌─────────────────┴───────────────────┐
                    │                                     │
                    ▼                                     ▼
             ┌─────────────┐                       ┌─────────────┐
             │  slurmctld  │                       │  slurmdbd   │
             └─────────────┘                       └─────────────┘
```

### 認證流程

```
客戶端 ──▶ 代理 ──▶ slurmrestd ──▶ slurmctld/slurmdbd

1. 客戶端向代理認證（SSO 等）
2. 代理設定 X-SLURM-USER-NAME 和 X-SLURM-USER-TOKEN
3. slurmrestd 驗證 JWT token
4. 請求轉發到 slurmctld/slurmdbd
```

---

## 實務範例

### 基本 slurmrestd 設定

```bash
# 使用監聽模式啟動
slurmrestd -a rest_auth/jwt 0.0.0.0:6820

# 使用 UNIX socket
slurmrestd unix:/var/run/slurmrestd.socket
```

### JWT 設定

```
# slurm.conf
AuthAltTypes=auth/jwt
AuthAltParameters=jwt_key=/etc/slurm/jwt_hs256.key

# slurmdbd.conf
AuthAltTypes=auth/jwt
```

### NGINX 代理設定

```nginx
upstream slurmrestd {
    server 127.0.0.1:6820;
}

server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://slurmrestd;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 直接暴露 slurmrestd 到網際網路 | 使用 TLS 代理 |
| 未設定快取代理 | 設定快取以減少控制器負載 |
| JWT token 期限過長 | 使用短期 token 並經常更新 |
| 以 root 運行 slurmrestd | 使用無特權使用者 |

### 建議

1. **安全性**：
   - 始終使用 TLS 代理
   - 實施速率限制
   - 監控認證失敗

2. **效能**：
   - 設定快取代理
   - 限制連線數
   - 監控 slurmctld 負載

3. **高可用性**：
   - 部署多個 slurmrestd 實例
   - 使用負載平衡器
   - 監控健康狀態

---

## 快速參考

### slurmrestd 選項

| 選項 | 說明 |
|------|------|
| `-a <auth_type>` | 認證類型 |
| `-t <count>` | 執行緒數 |
| `--max-connections <count>` | 最大連線數 |

### HTTP 標頭

| 標頭 | 用途 |
|------|------|
| X-SLURM-USER-NAME | 使用者名稱 |
| X-SLURM-USER-TOKEN | JWT token |

### 相關文件

- [REST API 快速入門](rest_quickstart.md) - 快速入門指南
- [JWT 認證](jwt.md) - JWT 設定
- [slurmrestd man page](slurmrestd.html) - 完整參考

