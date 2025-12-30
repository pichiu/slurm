# Slurm OpenAPI 外掛程式發行說明

## TL;DR

此文件提供 OpenAPI 客戶端程式設計師關於 OpenAPI 規格變更的指南。變更主要由 slurmrestd 使用，也用於產生多個 CLI 命令的 JSON 或 YAML 輸出。每個 Slurm 主要版本都會新增一個新的 data_parser 版本。詳細變更請查閱英文原始文件或產生的 OAS。

---

## 翻譯

### 概觀

這些發行說明旨在為 OpenAPI 客戶端程式設計師提供關於 OpenAPI 外掛程式產生的 OpenAPI 規格變更的指南。這些外掛程式主要由 [slurmrestd](rest.md) 使用，但也用於產生多個 CLI 命令的 JSON 或 YAML 輸出。

所有給定的路徑都是為 [jq](https://stedolan.github.io/jq/) 格式化的。確保始終將路徑放在單引號字串中以避免 shell 替換。

OpenAPI 規格應在完全設定正常操作後由 slurmrestd 產生。從 slurmrestd 查詢 `GET /openapi/v3` 以取得產生的 OpenAPI 規格。產生的規格可能會根據載入的外掛程式及其設定方式而改變。任何客戶端都必須小心始終使用目標 slurmrestd 守護程式的當前產生規格。客戶端開發應始終設計為使用最高版本的可用外掛程式，以避免比其他情況更快需要移植客戶端。

---

### 版本歷史

以下是各 Slurm 版本的 data_parser 外掛程式對照：

| Slurm 版本 | 新增 data_parser | 說明 |
|-----------|------------------|------|
| 20.02 | 初始版本 | REST API 首次引入 |
| 20.11 | - | 外掛程式架構改進 |
| 21.08 | - | OpenAPI 外掛程式可外部使用 |
| 22.05 | v0.0.38 | - |
| 23.02 | v0.0.39 | - |
| 23.11 | v0.0.40 | - |
| 24.05 | v0.0.41 | - |
| 24.11 | v0.0.42 | - |
| 25.05 | v0.0.43 | - |
| 25.11 | v0.0.44 | - |

---

### 查看詳細變更

由於 OpenAPI 規格的變更非常詳細且技術性強，建議開發者：

1. **查閱英文原始文件**：詳細的 API 變更列表請參閱 [openapi_release_notes.shtml](https://slurm.schedmd.com/openapi_release_notes.html)

2. **直接比較 OAS**：
```bash
# 產生兩個版本的 OAS
slurmrestd --generate-openapi-spec -d v0.0.43 > v43.json
slurmrestd --generate-openapi-spec -d v0.0.42 > v42.json

# 比較差異
diff v42.json v43.json
```

3. **使用 jq 查詢特定 schema**：
```bash
# 查看特定模型
jq '.components.schemas."v0.0.43_job"' openapi.json
```

---

### 變更類型

OpenAPI 規格的變更通常包括：

| 變更類型 | 說明 | 影響 |
|----------|------|------|
| 新增欄位 | 新增 API 回應中的欄位 | 通常無破壞性 |
| 移除欄位 | 從 API 回應中移除欄位 | 可能影響客戶端 |
| 類型變更 | 欄位資料類型變更 | 可能需要更新客戶端 |
| 列舉值變更 | 新增或移除列舉選項 | 取決於使用方式 |
| 路徑變更 | API 端點路徑變更 | 需要更新客戶端 |

---

### 處理升級

當升級 Slurm 版本時：

1. **確認外掛程式版本**：
   - 檢查當前使用的 data_parser 版本是否仍受支援
   - 參考[生命週期表](rest_clients.md)

2. **比較規格差異**：
```bash
# 遮罩版本標記以便比較
cat new_version.json | sed -e 's#v0.0.44#v0.0.43#g' > new_masked.json
diff old_version.json new_masked.json
```

3. **測試客戶端**：
   - 在升級前在測試環境中驗證客戶端
   - 檢查所有使用的 API 端點

---

## 說明

### 外掛程式架構

```
┌─────────────────────────────────────────────────────────────┐
│                        slurmrestd                            │
├─────────────────────────────────────────────────────────────┤
│                     OpenAPI 外掛程式                         │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │ openapi/slurmctld│  │ openapi/slurmdbd│                   │
│  └────────┬────────┘  └────────┬────────┘                   │
│           │                    │                             │
│           ▼                    ▼                             │
│  ┌─────────────────────────────────────┐                    │
│  │         data_parser 外掛程式         │                    │
│  ├──────────┬──────────┬──────────────┤                    │
│  │ v0.0.42  │ v0.0.43  │   v0.0.44    │                    │
│  └──────────┴──────────┴──────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 實務範例

### 產生發行說明

```bash
# 產生兩個版本的 OAS
OLD_VER=v0.0.42
NEW_VER=v0.0.43

slurmrestd --generate-openapi-spec -d $OLD_VER > old.json
slurmrestd --generate-openapi-spec -d $NEW_VER > new.json

# 遮罩版本進行比較
cat new.json | sed -e "s#$NEW_VER#$OLD_VER#g" > new_masked.json

# 使用 jsondiff 或類似工具
jsondiff old.json new_masked.json
```

### 追蹤特定 schema 變更

```bash
# 追蹤作業 schema 變更
jq '.components.schemas | keys | map(select(startswith("v0.0.43_job")))' openapi.json
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 忽略發行說明 | 每次升級前檢視相關變更 |
| 不測試就升級 | 在測試環境驗證客戶端 |
| 使用過時的外掛程式版本 | 使用最新穩定版本 |

### 建議

1. **訂閱發行說明**：
   - 追蹤 Slurm 發行說明
   - 關注 OpenAPI 相關變更

2. **自動化測試**：
   - 建立 API 回歸測試
   - 在 CI/CD 中包含 API 測試

3. **文件更新**：
   - 記錄使用的 API 版本
   - 維護變更日誌

---

## 快速參考

### 產生 OAS 命令

```bash
slurmrestd --generate-openapi-spec -s slurmctld,slurmdbd -d <version>
```

### jq 常用查詢

| 查詢 | 說明 |
|------|------|
| `.info.version` | API 版本 |
| `.paths | keys` | 所有路徑 |
| `.components.schemas | keys` | 所有 schema |

### 相關文件

- [REST API 客戶端指南](rest_clients.md) - 客戶端開發
- [REST API 詳細說明](rest.md) - slurmrestd 設定
- [升級指南](upgrades.md) - 版本升級

