# Slurm Elasticsearch 指南

## TL;DR

jobcomp/elasticsearch 外掛程式將已完成作業的資訊索引到 Elasticsearch，可搭配 Kibana 進行視覺化分析。設定 `JobCompType=jobcomp/elasticsearch` 和 `JobCompLoc=<host>:<port>/<index>/_doc`。需要 libcurl 和 JSON-C 函式庫。外掛程式會自動建立索引，並在 slurmctld 關閉時保存未索引的作業狀態。

---

## 翻譯

### 概觀

Slurm 提供多種作業完成外掛程式 (Job Completion Plugins)。這些外掛程式是為已完成作業提供歷史作業[計費](accounting.md)資料的正交方式。

在大多數安裝中，Slurm 已設定了 [AccountingStorageType](slurm.conf.html#OPT_AccountingStorageType) 外掛程式 — 通常是 **slurmdbd**。在這些情況下，完成外掛程式擷取的資訊是刻意冗餘的。

**jobcomp/elasticsearch** 外掛程式可與 Elasticsearch 伺服器上的 Web 層搭配使用 — 如 [Kibana](https://www.elastic.co/products/kibana) — 來視覺化已完成的作業和叢集狀態。這些視覺化工具還可讓您輕鬆建立不同類型的儀表板、圖表、表格、直方圖和/或在搜尋時套用自訂篩選器。

---

### 先決條件

此外掛程式需要額外的函式庫進行編譯：

| 函式庫 | 說明 |
|--------|------|
| [libcurl](https://curl.se/libcurl) | libcurl 開發檔案 |
| [JSON-C](related_software.html#json) | JSON 處理函式庫 |

---

### 設定

Elasticsearch 實例應該正在執行並可從設定的多個 [SlurmctldHost](slurm.conf.html#OPT_SlurmctldAddr) 存取。詳細的設定和配置請參閱 [Elasticsearch 官方文件](https://www.elastic.co/)。

#### slurm.conf 選項

**1. JobCompType**

用於選擇要啟用的作業完成外掛程式類型：

```
JobCompType=jobcomp/elasticsearch
```

**2. JobCompLoc**

設定為 Elasticsearch 伺服器 URL 端點（包括埠號和目標索引）：

```
JobCompLoc=<host>:<port>/<target>/_doc
```

**範例**：
```
JobCompLoc=localhost:9200/slurm/_doc
```

**注意事項**：

- 自 Elasticsearch 8.0 起，接受類型的 API 已移除，因此改為無類型模式
- Slurm 20.11 之前的版本會移除此選項 URL 的尾部斜線並附加硬編碼的 `/slurm/jobcomp` 後綴（代表 `/index/type`）
- 從 Slurm 20.11 開始，URL 完全可設定，不經修改直接傳遞給 libcurl 函式庫
- 這也允許使用者將不同叢集的資料索引到同一伺服器但不同索引

**3. DebugFlags**

可包含 **Elasticsearch** 旗標用於額外除錯：

```
DebugFlags=Elasticsearch
```

建議在驗證已完成作業正確索引之前先開啟此選項。

**注意**：您不需要手動建立 Elasticsearch 索引，因為外掛程式會在嘗試索引第一個作業文件時自動建立。

---

### 視覺化

作業開始被索引後，建議使用 Web 視覺化層來分析資料。

**[Kibana](https://www.elastic.co/products/kibana)** 是推薦的 Elasticsearch 開源資料視覺化外掛程式。

安裝後，需要設定 Elasticsearch 索引名稱或模式以指示 Kibana 擷取資料。資料載入後，可以：

- 建立表格，每行是一個已完成作業
- 按任意欄位排序（建議使用 @end_time 時間戳記）
- 建立任何儀表板、圖表或其他分析

---

### 測試與除錯

可使用 **curl** 命令或類似工具直接對 Elasticsearch 執行 REST 請求進行除錯。

#### 查詢索引資訊

```bash
$ curl -XGET http://localhost:9200/_cat/indices/slurm?v
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   slurm 103CW7GqQICiMQiSQv6M_g   5   1          9            0    142.8kb        142.8kb
```

#### 查詢所有索引的作業

```bash
$ curl -XGET 'http://localhost:9200/slurm/_search?pretty=true&q=*:*' | less
```

#### 刪除索引（注意！）

```bash
$ curl -XDELETE http://localhost:9200/slurm
{"acknowledged":true}
```

#### 查詢 _cat 選項

```bash
$ curl -XGET http://localhost:9200/_cat
```

更多資訊請參閱官方文件。

---

### 故障管理

當主要 slurmctld 關閉時，Elasticsearch 外掛程式中保存的所有已完成但尚未索引的作業資訊會儲存到名為 **elasticsearch_state** 的檔案中，位於 [StateSaveLocation](slurm.conf.html#OPT_StateSaveLocation)。

這允許外掛程式在 slurmctld 重新啟動時恢復資訊，並在連接恢復時傳送到 Elasticsearch 資料庫。

---

## 說明

### Elasticsearch 與 Slurm 整合架構

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  slurmctld  │────▶│ Elasticsearch│────▶│   Kibana    │
│             │     │   Server    │     │ (視覺化)    │
└─────────────┘     └─────────────┘     └─────────────┘
      │                    │
      │ 作業完成時         │ 索引資料
      │ 傳送文件           │
      ▼                    ▼
 jobcomp/         REST API
 elasticsearch    (JSON)
```

### 資料流程

1. 作業完成時，slurmctld 呼叫 jobcomp/elasticsearch
2. 外掛程式將作業資訊格式化為 JSON
3. 透過 libcurl 傳送到 Elasticsearch
4. Kibana 從 Elasticsearch 讀取並視覺化

---

## 實務範例

### 基本設定

```
# slurm.conf
JobCompType=jobcomp/elasticsearch
JobCompLoc=elasticsearch.example.com:9200/slurm/_doc

# 初始除錯（驗證後可移除）
DebugFlags=Elasticsearch
```

### 多叢集索引

```
# 叢集 A
JobCompLoc=elasticsearch.example.com:9200/cluster_a/_doc

# 叢集 B
JobCompLoc=elasticsearch.example.com:9200/cluster_b/_doc
```

### 驗證設定

```bash
# 檢查索引是否已建立
curl -XGET 'http://localhost:9200/_cat/indices?v'

# 檢查最近的作業
curl -XGET 'http://localhost:9200/slurm/_search?pretty&size=5&sort=@end_time:desc'

# 查看索引的欄位映射
curl -XGET 'http://localhost:9200/slurm/_mapping?pretty'
```

### Kibana 設定步驟

1. 安裝並啟動 Kibana
2. 存取 Kibana Web 介面（預設 http://localhost:5601）
3. 建立索引模式：`slurm*`
4. 選擇時間欄位：`@end_time`
5. 開始探索資料

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未安裝 libcurl 開發檔案 | 安裝 libcurl-devel 或 libcurl4-openssl-dev |
| JobCompLoc 格式錯誤 | 使用完整 URL 包含 /_doc 後綴 |
| Elasticsearch 無法連接 | 檢查防火牆和網路設定 |
| 舊版 Slurm 升級問題 | 參考官方文件重新索引資料 |

### 建議

1. **初始設定**：
   - 先開啟 DebugFlags=Elasticsearch
   - 驗證作業正確索引後再關閉

2. **效能考量**：
   - Elasticsearch 應有足夠資源
   - 考慮索引保留策略

3. **監控**：
   ```bash
   # 定期檢查索引狀態
   curl -XGET 'http://localhost:9200/_cluster/health?pretty'
   ```

---

## 快速參考

### slurm.conf 設定

```
JobCompType=jobcomp/elasticsearch
JobCompLoc=<host>:<port>/<index>/_doc
DebugFlags=Elasticsearch  # 可選，用於除錯
```

### 常用 curl 命令

| 命令 | 功能 |
|------|------|
| `curl -XGET 'http://host:9200/_cat/indices?v'` | 列出所有索引 |
| `curl -XGET 'http://host:9200/slurm/_search?pretty&q=*:*'` | 查詢所有作業 |
| `curl -XGET 'http://host:9200/slurm/_count'` | 計算文件數量 |
| `curl -XDELETE 'http://host:9200/slurm'` | 刪除索引 |

### 狀態檔案

| 檔案 | 位置 | 說明 |
|------|------|------|
| elasticsearch_state | StateSaveLocation | 未索引作業的暫存狀態 |

### 相關文件

- [計費](accounting.md) - 計費系統設定
- [作業完成 Kafka 外掛程式](jobcomp_kafka.md) - Kafka 整合
