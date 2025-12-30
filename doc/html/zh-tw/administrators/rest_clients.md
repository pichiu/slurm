# Slurm REST API 客戶端撰寫指南

## TL;DR

slurmrestd 符合 OpenAPI 3.0.2 標準。使用 `slurmrestd --generate-openapi-spec` 產生規格。客戶端應針對特定版本標記路徑設計。data_parser 外掛程式有明確的版本生命週期（每個版本支援約 2 年）。使用 OpenAPI 工具產生客戶端程式碼，建議將產生的程式碼放入版本控制。

---

## 翻譯

### OpenAPI 規格 (OAS)

slurmrestd 符合 [OpenAPI 3.0.2](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md)。產生的 OAS 可在以下 URL 查看：

- `/openapi.json`
- `/openapi.yaml`
- `/openapi/v3`

**產生 OAS 的方法**：

僅編譯 slurmrestd：
```bash
env SLURM_CONF=/dev/null slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d v0.0.40
```

完整設定的 Slurm 安裝：
```bash
slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d v0.0.40
```

OAS 設計為請求方法和回應（包括其內容）的主要文件。有多個第三方工具可自動產生 OAS 輸出的文件，列於 [openapi.tools](https://openapi.tools/)。

產生的 OpenAPI 規格會根據 slurmrestd 的執行時設定而改變。slurmrestd 是一個框架，實際內容由外掛程式提供，這些外掛程式在執行時是可選的。然而，特定的外掛程式版本（如路徑中的 v0.0.XX 所示）在 Slurm 版本之間將保持穩定。

---

### OpenAPI 標準符合性

Slurm 嘗試嚴格符合相關的 [OpenAPI 標準](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.2.md)。為了相容性，Slurm 的 OAS 會針對公開可用的 OpenAPI 客戶端產生器進行測試，但 Slurm 不直接支援任何特定產生器。目標是符合標準，盡可能支援更多客戶端，而不偏向任何一個。

**測試過的相容性**：

| 外掛程式 | 測試工具 |
|---------|----------|
| openapi/slurmctld | OpenAPI-generator v7.3.0, oapi-codegen v2.4.1 |
| openapi/slurmdbd | OpenAPI-generator v7.3.0, oapi-codegen v2.4.1 |

---

### OpenAPI 規格文件

Slurm 包含[範例產生文件](rest_api.md)，每個版本都會提供。Slurm 的文件只包含最新的外掛程式，以鼓勵站點針對最新外掛程式開發，因為它們的壽命最長。

**使用 OpenAPI-generator 產生 HTML 文件**：

1. 產生 OAS：
```bash
env SLURM_CONF=/dev/null slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d v0.0.40 > openapi.json
```

2. 產生文件：
```bash
openapi-generator-cli generate -i openapi.json -g html -o rest_api_docs
```

3. 用瀏覽器開啟 `rest_api_docs/index.html`

Swagger 提供 [Web 編輯器](https://editor-next.swagger.io/) 來查看和互動產生的 OAS。

---

### 客戶端設計

客戶端應始終針對或設計為使用 slurmrestd 標記路徑的特定版本。強烈建議客戶端開發者不要包含非預期使用版本的方法功能。

**版本規劃**：

如果計劃支援多個 Slurm 版本，客戶端開發者需要規劃如何優雅地處理不同版本之間的變更。Slurm 的版本控制方法是明確允許舊程式碼在該舊版本仍受支援時繼續與較新的 Slurm 版本一起工作。

例如，v0.0.38 方法在 Slurm-22.05 中新增，但可以使用到 Slurm-24.05。雖然這可行，但這些方法不會獲得任何新功能，通常只有安全修復。

**建議做法**：

- 建議使用僅針對一個版本產生的 OpenAPI schema
- 許多 OpenAPI 客戶端產生器有方法從結構名稱中去除版本標記（如 `V0039AccountFlagsDELETED` → `AccountFlagsDELETED`）
- 考慮在客戶端的程式碼庫中放置一組相對靜態的編譯客戶端程式碼
- 這將消除終端使用者執行程式碼產生器的需要

**OAS 變更注意事項**：

產生的 OpenAPI 規格可以改變，取決於存在哪些外掛程式，但版本化的路徑及其 schema 不會改變（有限例外）。產生僅限於 `v0.0.40` 的 schema 並放入您的程式碼庫，應該會產生可在 Slurm-23.11 到 Slurm-25.05 中使用的 schema。

---

### OpenAPI 規格變更

變更 OAS 時，每個版本都會在 [OpenAPI 發行說明](openapi_release_notes.md) 中列出。

**比較版本差異的技巧**：

```bash
env SLURM_CONF=/dev/null slurmrestd -d v0.0.41 -s slurmdbd,slurmctld --generate-openapi-spec > /tmp/v41.json
env SLURM_CONF=/dev/null slurmrestd -d v0.0.40 -s slurmdbd,slurmctld --generate-openapi-spec > /tmp/v40.json
cat /tmp/v41.json | sed -e 's#v0.0.41#v0.0.40#g' > /tmp/v41_masked.json
vimdiff /tmp/v40.json /tmp/v41_masked.json
```

**選擇特定 schema 進行比較**：
```bash
jq '.components.schemas."v0.0.40_job"' /tmp/v40.json > /tmp/v40_job.json
jq '.components.schemas."v0.0.40_openapi_job_info_resp".properties.jobs.items' /tmp/v41_masked.json > /tmp/v41_masked_job.json
vimdiff /tmp/v40_job.json /tmp/v41_masked_job.json
```

---

### Data Parser 生命週期

| data_parser 外掛程式 | 新增版本 | 移除版本 |
|---------------------|----------|----------|
| v0.0.39 | 23.02 | 24.11 |
| v0.0.40 | 23.11 | 25.11 |
| v0.0.41 | 24.05 | 26.05 |
| v0.0.42 | 24.11 | 26.11 |
| v0.0.43 | 25.05 | 27.05 |
| v0.0.44 | 25.11 | 27.11 |

data_parser 外掛程式有明確的版本控制和計劃的發行/移除日期。在不同版本之間刻意保持相同，而是為每個主要版本新增一個遞增的版本。這允許開發者在 Slurm 升級後繼續使用相同的客戶端，只要該客戶端是為仍存在於新 Slurm 版本中的 data_parser 外掛程式編寫的。

**重要**：
- 站點始終鼓勵使用最新穩定的外掛程式版本進行程式碼開發
- 不同版本的相同外掛程式之間不保證與客戶端的相容性
- 遷移到較新版本的外掛程式時，客戶端應始終驗證其程式碼

---

### 處理格式變更

使用 CLI 命令的 `--json` 或 `--yaml` 參數的任何腳本或客戶端可能需要明確傳遞 data_parser 版本以避免升級後的問題。

**比較格式差異**：
```bash
$CLI_COMMAND --json=v0.0.41+spec_only > /tmp/v41.json
$CLI_COMMAND --json=v0.0.40+spec_only > /tmp/v40.json
json_diff /tmp/v40.json /tmp/v41.json
```

**明確請求首選 data_parser**：
```bash
$CLI_COMMAND --json=v0.0.41 $OTHER_ARGS | $SITE_SCRIPT
$CLI_COMMAND --json=v0.0.40 $OTHER_ARGS | $SITE_SCRIPT
$CLI_COMMAND --yaml=v0.0.41 $OTHER_ARGS | $SITE_SCRIPT
$CLI_COMMAND --yaml=v0.0.40 $OTHER_ARGS | $SITE_SCRIPT
```

**slurmrestd URL 範例**：
```
http://$HOST/slurmdb/v0.0.40/jobs
http://$HOST/slurm/v0.0.40/jobs
```

---

## 說明

### 版本生命週期圖

```
Slurm 版本:  23.02  23.11  24.05  24.11  25.05  25.11  26.05  26.11  27.05
              │      │      │      │      │      │      │      │      │
v0.0.39 ──────┼──────┼──────┼──────X
              │      │      │      │
v0.0.40 ──────┴──────┼──────┼──────┼──────┼──────X
                     │      │      │      │      │
v0.0.41 ─────────────┴──────┼──────┼──────┼──────┼──────X
                            │      │      │      │      │
v0.0.42 ────────────────────┴──────┼──────┼──────┼──────┼──────X
                                   │      │      │      │      │
v0.0.43 ───────────────────────────┴──────┼──────┼──────┼──────┼──────X
                                          │      │      │      │      │
v0.0.44 ──────────────────────────────────┴──────┼──────┼──────┼──────┼─────
```

### 客戶端開發流程

```
1. 產生 OAS
   slurmrestd --generate-openapi-spec > openapi.json
                    │
                    ▼
2. 產生客戶端程式碼
   openapi-generator generate -i openapi.json -g python -o client
                    │
                    ▼
3. 將客戶端程式碼加入版本控制
   git add client/
                    │
                    ▼
4. 開發應用程式
   使用產生的客戶端存取 slurmrestd
                    │
                    ▼
5. 升級時
   驗證外掛程式版本仍受支援
   必要時更新客戶端程式碼
```

---

## 實務範例

### 產生 Python 客戶端

```bash
# 產生 OAS
slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d v0.0.43 > openapi.json

# 產生 Python 客戶端
openapi-generator-cli generate \
  -i openapi.json \
  -g python \
  -o python_client \
  --additional-properties=packageName=slurm_client

# 安裝客戶端
cd python_client
pip install -e .
```

### 產生 Go 客戶端

```bash
# 產生 OAS
slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d v0.0.43 > openapi.json

# 使用 oapi-codegen
oapi-codegen -package slurm -generate types,client openapi.json > slurm_client.go
```

### 處理版本升級

```python
# 在客戶端程式碼中處理多版本
def get_jobs(client, api_version="v0.0.43"):
    if api_version == "v0.0.43":
        return client.slurm_v0043_get_jobs()
    elif api_version == "v0.0.42":
        return client.slurm_v0042_get_jobs()
    else:
        raise ValueError(f"Unsupported API version: {api_version}")
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 使用即將移除的外掛程式版本 | 檢查生命週期表，使用較新版本 |
| 每次都重新產生客戶端程式碼 | 將產生的程式碼放入版本控制 |
| 忽略版本相容性 | 升級前測試客戶端 |
| 使用未版本化的路徑 | 始終使用版本化的 API 路徑 |

### 建議

1. **版本策略**：
   - 使用最新穩定的外掛程式版本
   - 在生命週期結束前 2 個主要版本升級

2. **客戶端維護**：
   - 將產生的程式碼放入版本控制
   - 記錄使用的 OAS 版本
   - 在升級前徹底測試

3. **文件**：
   - 為團隊記錄 API 使用方式
   - 追蹤外掛程式版本變更

---

## 快速參考

### 產生 OAS 命令

```bash
slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d <version>
```

### 常用 OpenAPI 工具

| 工具 | 用途 |
|------|------|
| openapi-generator-cli | 產生多種語言客戶端 |
| oapi-codegen | Go 客戶端產生 |
| Swagger Editor | 線上查看/編輯 OAS |

### OAS 端點

| URL | 說明 |
|-----|------|
| /openapi.json | JSON 格式 OAS |
| /openapi.yaml | YAML 格式 OAS |
| /openapi/v3 | OpenAPI v3 規格 |

### 相關文件

- [REST API 詳細說明](rest.md) - slurmrestd 設定
- [REST API 快速入門](rest_quickstart.md) - 快速入門指南
- [JWT 認證](jwt.md) - 認證設定
- [升級指南](upgrades.md) - 版本升級

