# Slurm REST API 快速入門指南 (REST API Quick Start Guide)

## TL;DR

Slurm 透過 slurmrestd 守護程式提供 REST API，使用 JWT 進行認證。安裝 slurm-slurmrestd 套件、設定 JWT、執行 `slurmrestd :6820` 即可啟動。使用 `scontrol token` 獲取認證令牌，透過 curl 存取 API 端點如 `/slurm/v0.0.XX/diag`。預設監聽 TCP 6820 埠，也可使用 UNIX socket。

---

## 翻譯

### 概觀

Slurm 透過 slurmrestd 守護程式提供 [REST API](https://restfulapi.net/)，使用 [JSON Web Tokens](jwt.md) 進行認證。此守護程式旨在允許客戶端透過 REST API（除了命令列介面 (CLI) 或 C API 外）與 Slurm 通訊。

**相關文件**：
- [REST API 詳細資訊](rest.md)
- [REST API 方法和模型](rest_api.html)
- [slurmrestd man page](slurmrestd.html)
- [OpenAPI 外掛程式發行說明](openapi_release_notes.md)
- [REST API 客戶端指南](rest_clients.md)

---

### 先決條件

#### 編譯時必要的開發函式庫

| 函式庫 | 說明 |
|--------|------|
| HTTP Parser | HTTP 解析器 |
| JSON-C | JSON 處理 |

#### 選用的開發函式庫

| 函式庫 | 功能 |
|--------|------|
| YAML | YAML 支援 |
| JWT | socket 客戶端認證 |
| s2n | TLS 通訊 |

**建議**：設定好 [slurmdbd](slurmdbd.html) 進行[計費](accounting.md)。若沒有 slurmdbd，可能需要使用 `-s` 旗標啟動 slurmrestd 以告訴它不要載入 slurmdbd 外掛程式。

---

### 快速開始

可在專用的 REST API 機器或現有的 slurmctld 機器上執行，視需求而定。如果有多個叢集，每個叢集需要一個獨立的 slurmrestd 實例。

#### 安裝步驟

1. **安裝 slurmrestd 元件**
   ```bash
   # DEB
   apt install slurm-smd slurm-smd-slurmrestd

   # RPM（建置時需要 --with slurmrestd）
   yum install slurm slurm-slurmrestd
   ```

2. **設定 JSON Web Tokens 認證**
   - 參考 [JWT 認證](jwt.md)

3. **確保 slurm.conf 正確**
   - 參考[快速入門管理指南](quickstart_admin.md)和 [slurm.conf man page](slurm.conf.html)

4. **執行 slurmrestd**
   ```bash
   export SLURM_JWT=daemon
   export SLURMRESTD_DEBUG=debug
   slurmrestd <host>:<port>
   ```
   預設生產環境使用 `:6820`。

---

### 使用 systemd 執行

Slurm 附帶 slurmrestd 的 systemd 服務單元，但可能需要一些額外設定。

#### 設定步驟

1. **建立服務帳戶**
   ```bash
   sudo useradd -M -r -s /usr/sbin/nologin -U slurmrestd
   ```

   **帳戶要求**：
   - 不是 root 或 SlurmUser
   - 不被真實使用者使用或用於其他功能
   - 不授予任何特殊權限

2. **覆寫使用者和群組（如需要）**
   ```bash
   systemctl edit slurmrestd
   ```
   ```ini
   [Service]
   User=slurmrestd
   Group=slurmrestd
   ```

3. **設定監聽 socket**

   **Slurm 24.05+**：修改 `SLURMRESTD_LISTEN` 環境變數

   **Slurm 23.11 及更早版本**：
   ```bash
   systemctl edit slurmrestd
   ```
   ```ini
   [Service]
   ExecStart=
   ExecStart=/usr/sbin/slurmrestd $SLURMRESTD_OPTIONS
   Environment=SLURMRESTD_LISTEN=:6820
   ```

---

### 自訂 slurmrestd.service

Slurm 24.05 版本變更了預設服務檔案的運作方式。如果您覆寫了 `ExecStart=` 直接包含 TCP/UNIX sockets，且與 SLURMRESTD_LISTEN 中的 sockets 重複，服務將會失敗。

#### 自訂方式

**1. 環境檔案**

服務會從以下檔案讀取環境變數：
- `/etc/default/slurmrestd`
- `/etc/sysconfig/slurmrestd`

| 變數 | 說明 |
|------|------|
| SLURMRESTD_OPTIONS | 新增到 slurmrestd 命令的 CLI 選項 |
| SLURMRESTD_LISTEN | 逗號分隔的 host:port 對或 unix:$SOCKET_PATH |

**2. 服務編輯**

```bash
systemctl edit slurmrestd
```

建立覆寫檔案在 `/etc/systemd/` 中。變更可透過以下命令還原：
```bash
systemctl revert slurmrestd
```

---

### 基本使用

1. **查詢支援的 API 版本**
   ```bash
   slurmrestd -d list
   ```

2. **獲取 JWT 認證令牌**
   ```bash
   unset SLURM_JWT; export $(scontrol token)
   ```
   - 確保舊令牌不會阻止新令牌發行
   - 預設令牌在 1800 秒（30 分鐘）後過期
   - 新增 `lifespan=SECONDS` 以變更有效期

3. **使用 TCP host:port 執行 curl**
   ```bash
   curl -s -o "/tmp/curl.log" -k -vvvv \
   -H X-SLURM-USER-TOKEN:$SLURM_JWT \
   -X GET 'http://<server>:<port>/slurm/v0.0.<api-version>/diag'
   ```
   - 替換 server、port 和 api-version
   - 確認回應為 200 OK
   - 檢查 /tmp/curl.log 中的 JSON 回應

4. **使用 UNIX socket 的替代命令**
   ```bash
   curl -s -o "/tmp/curl.log" -k -vvvv \
   -H X-SLURM-USER-TOKEN:$SLURM_JWT \
   --unix-socket /path/to/slurmrestd.socket \
   'http://<server>/slurm/v0.0.<api-version>/diag'
   ```

#### 令牌管理

本指南提供使用 `scontrol` 獲取令牌的簡單概觀。這是基本入門方法，在許多情況下應停用以採用更複雜的令牌管理。詳情請參考 [JWT 頁面](jwt.md)。

---

### 進階使用

關於進一步自訂和設定 slurmrestd 的資訊，包括認證方法、執行模式、外掛程式、高可用性、代理和 Python 客戶端，請參閱 [REST API 詳細資訊](rest.md)頁面。

---

### 常見問題

一般而言，請注意以下事項：
1. `SLURM_JWT` 中認證令牌的有效性
2. 主機名稱和埠號
3. API 版本和端點
4. slurmrestd 的日誌輸出

---

## 常見錯誤與建議

### 無法綁定 socket

**原因**：設定 socket 時的權限問題

**解決方案**：
- 檢查 slurmrestd 日誌輸出中的 socket 路徑
- 確保執行 slurmrestd 服務的使用者對 socket 路徑的父目錄有寫入+執行權限
- 或變更/移除 socket 路徑

**「Address already in use」**：
- 檢查執行的命令和 SLURMRESTD_LISTEN 內容是否有重複的 TCP 或 UNIX socket

### 連接被拒絕

驗證 slurmrestd 正在執行並監聽您嘗試連接的埠。

### 協定認證錯誤 (HTTP 500)

**常見原因**：令牌過期

**解決方案**：
```bash
unset SLURM_JWT; export $(scontrol token)
```

此解決方案也適用於因未發送認證令牌而導致的 HTTP 401 錯誤。slurmrestd 日誌中可能顯示「Authentication does not apply to request.」

否則，請查閱 slurmctld 和 slurmdbd 的日誌。

### 找不到請求的 URL (HTTP 404)

檢查 [API 方法和模型](rest_api.html) 頁面，確保使用有效的 URL 和正確的方法。注意路徑，slurm 和 slurmdbd 有不同的端點。

### 拒絕執行緒設定令牌 (HTTP 401)

**檢查** slurmrestd 是否載入了 auth/jwt 外掛程式：
```
slurmrestd: debug:  auth/jwt: init: JWT authentication plugin loaded
```

**如果未載入**：
```bash
export SLURM_JWT=daemon
```

### 意外的 URL 字元 (HTTP 400)

使用適當的 URL 編碼序列替換有問題的字元：

```bash
# 錯誤
curl -X GET "localhost:8080/slurmdb/v0.0.40/jobs?submit_time=02/28/24"
### 400 BAD REQUEST

# 正確（/ = %2F）
curl -X GET "localhost:8080/slurmdb/v0.0.40/jobs?submit_time=02%2F28%2F24"
### 200 OK
```

### 其他 Slurm 命令無法運作

如果設定了 SLURM_JWT，其他 Slurm 命令會嘗試使用 JWT 認證，導致失敗。

**解決方案**：
```bash
unset SLURM_JWT
```

---

## 實務範例

### 完整設定流程

```bash
# 1. 安裝
yum install slurm slurm-slurmrestd

# 2. 建立服務帳戶
useradd -M -r -s /usr/sbin/nologin -U slurmrestd

# 3. 設定 JWT（參考 jwt.md）

# 4. 啟動服務
systemctl start slurmrestd
systemctl enable slurmrestd
```

### 環境檔案設定

```bash
# /etc/sysconfig/slurmrestd
SLURMRESTD_LISTEN=:6820
SLURMRESTD_OPTIONS="-v"
```

### 測試 API

```bash
# 獲取令牌
unset SLURM_JWT
export $(scontrol token)

# 測試 diag 端點
curl -H "X-SLURM-USER-TOKEN:$SLURM_JWT" \
     http://localhost:6820/slurm/v0.0.41/diag | jq

# 列出作業
curl -H "X-SLURM-USER-TOKEN:$SLURM_JWT" \
     http://localhost:6820/slurm/v0.0.41/jobs | jq

# 提交作業
curl -H "X-SLURM-USER-TOKEN:$SLURM_JWT" \
     -H "Content-Type: application/json" \
     -X POST \
     -d '{"job": {"name": "test", "script": "#!/bin/bash\necho hello"}}' \
     http://localhost:6820/slurm/v0.0.41/job/submit
```

---

## 快速參考

### 安裝命令

```bash
# DEB
apt install slurm-smd-slurmrestd

# RPM
yum install slurm-slurmrestd
```

### 服務管理

```bash
# 啟動
systemctl start slurmrestd

# 檢查狀態
systemctl status slurmrestd

# 查看日誌
journalctl -u slurmrestd -f

# 編輯設定
systemctl edit slurmrestd
```

### 環境變數

| 變數 | 說明 |
|------|------|
| SLURM_JWT | 設為 `daemon` 以啟用 JWT 認證 |
| SLURMRESTD_DEBUG | 除錯等級 |
| SLURMRESTD_LISTEN | 監聽位址 |
| SLURMRESTD_OPTIONS | CLI 選項 |

### 常用端點

| 端點 | 方法 | 說明 |
|------|------|------|
| `/slurm/v0.0.XX/diag` | GET | 診斷資訊 |
| `/slurm/v0.0.XX/jobs` | GET | 列出作業 |
| `/slurm/v0.0.XX/job/submit` | POST | 提交作業 |
| `/slurm/v0.0.XX/nodes` | GET | 列出節點 |
| `/slurmdb/v0.0.XX/jobs` | GET | 資料庫作業資訊 |

### 相關文件

- [JWT 認證](jwt.md) - JWT 設定
- [REST API 詳細資訊](rest.md) - 進階設定
- [REST API 方法和模型](rest_api.html) - API 參考
- [計費](accounting.md) - 計費設定
