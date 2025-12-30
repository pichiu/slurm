# Slurm 作業完成 Kafka 外掛程式指南

## TL;DR

jobcomp/kafka 外掛程式將已完成作業記錄發布到 Kafka 伺服器。設定 `JobCompType=jobcomp/kafka`，`JobCompLoc` 指向 librdkafka 設定檔。需要 librdkafka 和 libjson-c 函式庫。從 Slurm 25.05 起也可發送作業開始事件。外掛程式使用最少負載演算法，支援訊息重試和狀態持久化。

---

## 翻譯

### 概觀

設定後，**jobcomp/kafka** 外掛程式會嘗試將每個已完成作業記錄的欄位子集發布到 Kafka 伺服器。從 Slurm 25.05 起，也可以選擇在作業開始執行時發送作業記錄欄位的子集。

---

### 先決條件

外掛程式在每次發布嘗試前將欄位子集序列化為 JSON。序列化是使用 Slurm 序列化外掛程式完成的，因此 **libjson-c** 開發檔案是此外掛程式的間接先決條件。

外掛程式將部分客戶端生產者工作卸載到 **librdkafka** 並使用其 API，因此該函式庫的開發檔案是另一個先決條件。

| 函式庫 | 說明 |
|--------|------|
| [librdkafka](https://github.com/confluentinc/librdkafka) | Kafka 客戶端函式庫開發檔案 |
| [libjson-c](related_software.html#json) | JSON 處理函式庫開發檔案 |

---

### 設定

外掛程式透過以下 slurm.conf 選項設定：

#### JobCompType

應設定為 **jobcomp/kafka**：

```
JobCompType=jobcomp/kafka
```

#### JobCompLoc

此字串代表一個檔案的絕對路徑，該檔案包含設定 [librdkafka 屬性](https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md) 的 'key=value' 配對。

```
JobCompLoc=/arbitrary/path/to/rdkafka.conf
```

為了讓外掛程式正常運作，此檔案需要存在，且至少需要設定 **bootstrap.servers** 屬性。

**注意事項**：
- 設定此外掛程式時，JobCompLoc 沒有預設值，因此需要明確設定
- 此選項引用的檔案中設定的 **librdkafka** 參數在 slurmctld 重新啟動時生效
- 外掛程式不驗證這些參數，但如果傳遞給函式庫 API 函式 rd_kafka_conf_set() 的任何參數失敗，會記錄錯誤

**範例設定檔**：

```
bootstrap.servers=kafkahost1:9092
debug=broker,topic,msg
linger.ms=400
log_level=7
```

#### JobCompParams

以逗號分隔的額外可設定參數列表。詳細資訊請參閱 slurm.conf man page。

```
JobCompParams=flush_timeout=200,poll_interval=3,requeue_on_msg_timeout,topic=mycluster
```

**注意**：
- 對此選項的變更不需要 slurmctld 重新啟動。重新設定或 SIGHUP 足以使其生效
- 請參閱 man page 以設定作業開始事件

#### DebugFlags

可選的 **JobComp** 除錯旗標，用於額外的外掛程式特定記錄：

```
DebugFlags=JobComp
```

---

### 外掛程式功能

對於每個完成的（或可選開始執行的）作業，執行外掛程式 jobcomp_p_record_job_[end|start] 操作。作業記錄欄位的子集透過 Slurm 序列化外掛程式序列化為 JSON 字串。然後嘗試使用 librdkafka rd_kafka_producev() API 呼叫發布序列化的字串。

**訊息處理流程**：

1. 即使 Kafka 伺服器當機，也可以向 librdkafka 發布訊息
2. 如果此呼叫回傳錯誤，訊息會被丟棄
3. 發布的訊息在 librdkafka 輸出佇列中累積，最多累積 "linger.ms" 毫秒
4. 然後從累積的訊息建立訊息集
5. librdkafka 函式庫傳送發布請求

**回應處理**：

| 回應類型 | 處理方式 |
|----------|----------|
| 可重試錯誤 | 函式庫會自動嘗試重試（如果未達函式庫限制參數）|
| 永久錯誤或成功 | 訊息從函式庫輸出佇列移除，並暫存到函式庫傳遞報告佇列 |

---

### 背景輪詢處理器

**jobcomp/kafka** 外掛程式有一個背景輪詢處理器執行緒，定期呼叫 librdkafka API rd_kafka_poll() 函式。執行緒呼叫的頻率可透過 JobCompParams=poll_interval 設定。

此呼叫使函式庫傳遞報告佇列中的訊息被拉取並處理回外掛程式傳遞報告回呼，該回呼根據函式庫設定的錯誤訊息採取不同動作：

| 訊息類型 | 預設動作 |
|----------|----------|
| 成功訊息 | 如果啟用 DebugFlags=JobComp 則記錄 |
| 永久錯誤訊息 | 丟棄 |
| 訊息超時（且設定 requeue_on_msg_timeout）| 嘗試重新發布 |

---

### 外掛程式終止

在外掛程式終止時，執行 fini() 操作：

1. 呼叫 rd_kafka_purge() 函式庫 API，清除 librdkafka 輸出佇列訊息
2. 呼叫 rd_kafka_flush() API，等待所有未完成的發布請求完成
3. 等待時間可透過 JobCompParams=flush_timeout 參數設定
4. 清除的訊息始終儲存到 Slurm StateSaveLocation 中的外掛程式狀態檔
5. "in-flight" 中清除的訊息會被丟棄

**注意**：您必須使用 librdkafka v1.0.0 或更高版本才能利用上述清除功能。使用較舊版本時，輸出佇列無法在關閉時清除到狀態檔，這意味著在 kafka 外掛程式終止前未傳遞的任何訊息都會遺失。

---

### 外掛程式初始化

在外掛程式初始化時，解析設定後，會載入狀態中儲存的訊息並嘗試重新發布。因此未傳遞的訊息應該能夠在 slurmctld 重新啟動後恢復。

---

### 其他資訊

- Kafka broker "host:port" 列表應在 JobCompLoc 選項引用的檔案中明確設定
- 預設主題是設定的 Slurm ClusterName，但也可以透過 JobCompParams=topic 參數設定
- **jobcomp/kafka** 外掛程式主要將資訊訊息記錄到 JobComp DebugFlag，錯誤訊息除外
- librdkafka 預設記錄到應用程式 stderr，但外掛程式將函式庫設定為強制記錄到 syslog
- 函式庫記錄等級和除錯上下文也可透過 JobCompLoc 引用的檔案設定

---

## 說明

### 訊息流程架構

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  slurmctld  │────▶│ librdkafka  │────▶│   Kafka     │
│             │     │  (out queue) │     │   Broker    │
└─────────────┘     └──────┬──────┘     └─────────────┘
      │                    │
      │ 作業完成/開始       │ 定期 poll
      │                    ▼
      │              ┌──────────────┐
      │              │ delivery     │
      │              │ report queue │
      │              └──────────────┘
      │
      ▼
 狀態檔案
 (StateSaveLocation)
```

### 訊息生命週期

```
作業完成/開始
      │
      ▼
 序列化為 JSON
      │
      ▼
 rd_kafka_producev()
      │
      ├──▶ 成功 ──▶ 加入 out queue
      │                    │
      └──▶ 失敗 ──▶ 丟棄   │
                          ▼
                    等待 linger.ms
                          │
                          ▼
                    發送 produce 請求
                          │
                    ┌─────┴─────┐
                    ▼           ▼
               可重試錯誤    成功/永久錯誤
                    │           │
                    ▼           ▼
               自動重試     delivery report
                                │
                           ┌────┴────┐
                           ▼         ▼
                        成功      錯誤
                           │         │
                           ▼         ▼
                        記錄      丟棄/重試
```

---

## 實務範例

### 基本設定

```
# slurm.conf
JobCompType=jobcomp/kafka
JobCompLoc=/etc/slurm/rdkafka.conf
JobCompParams=topic=slurm_jobs,poll_interval=5
DebugFlags=JobComp
```

```
# /etc/slurm/rdkafka.conf
bootstrap.servers=kafka1:9092,kafka2:9092
linger.ms=500
log_level=6
```

### 高可用性設定

```
# /etc/slurm/rdkafka.conf
bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092
linger.ms=200
message.timeout.ms=30000
retries=5
retry.backoff.ms=500
```

### 作業開始事件設定

```
# slurm.conf（Slurm 25.05+）
JobCompType=jobcomp/kafka
JobCompLoc=/etc/slurm/rdkafka.conf
JobCompParams=topic=slurm_jobs,send_start_events
```

### 訊息重試設定

```
# slurm.conf
JobCompParams=flush_timeout=500,poll_interval=3,requeue_on_msg_timeout,topic=mycluster
```

### 驗證設定

```bash
# 檢查 Kafka 連接
kafka-console-consumer.sh --bootstrap-server kafka1:9092 --topic slurm_jobs

# 監控 slurmctld 記錄
tail -f /var/log/slurm/slurmctld.log | grep -i kafka
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 bootstrap.servers | 在 rdkafka.conf 中明確設定 |
| JobCompLoc 路徑錯誤 | 使用絕對路徑 |
| 使用舊版 librdkafka | 升級到 v1.0.0+ 以支援狀態持久化 |
| 設定變更後未重啟 | rdkafka.conf 變更需重啟 slurmctld |

### 建議

1. **效能調校**：
   - 調整 linger.ms 平衡延遲和吞吐量
   - 設定適當的 poll_interval
   - 監控訊息佇列大小

2. **可靠性**：
   - 使用多個 bootstrap.servers
   - 啟用 requeue_on_msg_timeout
   - 設定合理的 flush_timeout

3. **監控**：
   ```bash
   # 監控未傳遞訊息
   ls -la /var/spool/slurm/kafka_state

   # 檢查外掛程式狀態
   scontrol show config | grep JobComp
   ```

4. **除錯**：
   - 開發時啟用 DebugFlags=JobComp
   - 檢查 syslog 中的 librdkafka 訊息
   - 在 rdkafka.conf 中設定 debug=broker,topic,msg

---

## 快速參考

### slurm.conf 設定

```
JobCompType=jobcomp/kafka
JobCompLoc=/path/to/rdkafka.conf
JobCompParams=topic=name,poll_interval=N,flush_timeout=N,requeue_on_msg_timeout
DebugFlags=JobComp
```

### JobCompParams 參數

| 參數 | 說明 |
|------|------|
| topic | Kafka 主題名稱（預設為 ClusterName）|
| poll_interval | 輪詢間隔（秒）|
| flush_timeout | 關閉時等待時間（毫秒）|
| requeue_on_msg_timeout | 超時訊息重新排隊 |
| send_start_events | 發送作業開始事件（25.05+）|

### rdkafka.conf 常用設定

| 參數 | 說明 |
|------|------|
| bootstrap.servers | Kafka broker 列表 |
| linger.ms | 訊息批次等待時間 |
| log_level | 記錄等級（0-7）|
| debug | 除錯類別 |
| message.timeout.ms | 訊息超時 |
| retries | 重試次數 |

### 相關文件

- [Elasticsearch 指南](elasticsearch.md) - Elasticsearch 整合
- [計費](accounting.md) - 計費系統設定

