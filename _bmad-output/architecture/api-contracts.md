# Slurm REST API 契約

> 產生日期：2025-12-17 | 來源：slurmrestd OpenAPI 外掛

## 概覽

Slurm 透過 `slurmrestd` 守護程式提供完整的 REST API，提供 HTTP/HTTPS 存取叢集管理功能。API 分為兩個主要外掛：

1. **openapi/slurmctld** - 控制器操作（作業、節點、分割區）
2. **openapi/slurmdbd** - 資料庫操作（記帳、使用者、QoS）

**基本連接埠**：6820（可設定）
**驗證**：JWT 權杖或本機 Unix socket

---

## slurmctld API 端點（`/slurm/{data_parser}/...`）

### 作業

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurm/{data_parser}/jobs/` | 取得作業清單 |
| DELETE | `/slurm/{data_parser}/jobs/` | 發送訊號至作業清單 |
| GET | `/slurm/{data_parser}/jobs/state/` | 取得作業狀態清單 |
| GET | `/slurm/{data_parser}/job/{job_id}` | 取得作業資訊 |
| POST | `/slurm/{data_parser}/job/{job_id}` | 更新作業 |
| DELETE | `/slurm/{data_parser}/job/{job_id}` | 取消或發送訊號至作業 |
| POST | `/slurm/{data_parser}/job/submit` | 提交新作業 |
| POST | `/slurm/{data_parser}/job/allocate` | 提交新作業分配 |

### 節點

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurm/{data_parser}/nodes/` | 取得節點資訊 |
| POST | `/slurm/{data_parser}/nodes/` | 批次更新節點 |
| GET | `/slurm/{data_parser}/node/{node_name}` | 取得節點資訊 |
| POST | `/slurm/{data_parser}/node/{node_name}` | 更新節點屬性 |
| DELETE | `/slurm/{data_parser}/node/{node_name}` | 刪除節點 |
| POST | `/slurm/{data_parser}/new/node/` | 建立節點 |

### 分割區

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurm/{data_parser}/partitions/` | 取得所有分割區資訊 |
| GET | `/slurm/{data_parser}/partition/{partition_name}` | 取得分割區資訊 |

### 保留

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurm/{data_parser}/reservations/` | 取得所有保留資訊 |
| POST | `/slurm/{data_parser}/reservations/` | 建立或更新保留 |
| GET | `/slurm/{data_parser}/reservation/{reservation_name}` | 取得保留資訊 |
| POST | `/slurm/{data_parser}/reservation` | 建立或更新保留 |
| DELETE | `/slurm/{data_parser}/reservation/{reservation_name}` | 刪除保留 |

### 系統

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurm/{data_parser}/ping/` | Ping 測試 |
| GET | `/slurm/{data_parser}/diag/` | 取得診斷資訊 |
| GET | `/slurm/{data_parser}/reconfigure/` | 要求 slurmctld 重新設定 |
| GET | `/slurm/{data_parser}/licenses/` | 取得所有 Slurm 追蹤的授權資訊 |
| GET | `/slurm/{data_parser}/shares` | 取得公平共用資訊 |
| GET | `/slurm/{data_parser}/resources/{job_id}` | 取得資源配置資訊 |

---

## slurmdbd API 端點（`/slurmdb/{data_parser}/...`）

### 作業（記帳）

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/jobs/` | 取得作業清單 |
| POST | `/slurmdb/{data_parser}/jobs/` | 更新作業 |
| GET | `/slurmdb/{data_parser}/job/{job_id}` | 取得作業資訊 |
| POST | `/slurmdb/{data_parser}/job/{job_id}` | 更新作業 |

### 使用者

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/users/` | 取得使用者清單 |
| POST | `/slurmdb/{data_parser}/users/` | 更新使用者 |
| GET | `/slurmdb/{data_parser}/user/{name}` | 取得使用者資訊 |
| DELETE | `/slurmdb/{data_parser}/user/{name}` | 刪除使用者 |
| POST | `/slurmdb/{data_parser}/users_association/` | 新增使用者並條件式關聯 |

### 帳戶

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/accounts/` | 取得帳戶清單 |
| POST | `/slurmdb/{data_parser}/accounts/` | 新增/更新帳戶清單 |
| GET | `/slurmdb/{data_parser}/account/{account_name}` | 取得帳戶資訊 |
| DELETE | `/slurmdb/{data_parser}/account/{account_name}` | 刪除帳戶 |
| POST | `/slurmdb/{data_parser}/accounts_association/` | 新增帳戶並條件式關聯 |

### 關聯

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/associations/` | 取得關聯清單 |
| POST | `/slurmdb/{data_parser}/associations/` | 設定關聯資訊 |
| DELETE | `/slurmdb/{data_parser}/associations/` | 刪除關聯 |
| GET | `/slurmdb/{data_parser}/association/` | 取得關聯資訊 |
| DELETE | `/slurmdb/{data_parser}/association/` | 刪除關聯 |

### QoS（服務品質）

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/qos/` | 取得 QoS 清單 |
| POST | `/slurmdb/{data_parser}/qos/` | 新增或更新 QoS |
| GET | `/slurmdb/{data_parser}/qos/{qos}` | 取得 QoS 資訊 |
| DELETE | `/slurmdb/{data_parser}/qos/{qos}` | 刪除 QoS |

### 叢集

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/clusters/` | 取得叢集清單 |
| POST | `/slurmdb/{data_parser}/clusters/` | 修改叢集 |
| GET | `/slurmdb/{data_parser}/cluster/{cluster_name}` | 取得叢集資訊 |
| DELETE | `/slurmdb/{data_parser}/cluster/{cluster_name}` | 刪除叢集 |

### TRES（可追蹤資源）

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/tres/` | 取得 TRES 資訊 |
| POST | `/slurmdb/{data_parser}/tres/` | 新增 TRES |

### WCKey

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/wckeys/` | 取得 WCKey 清單 |
| POST | `/slurmdb/{data_parser}/wckeys/` | 新增或更新 WCKey |
| GET | `/slurmdb/{data_parser}/wckey/{id}` | 取得 WCKey 資訊 |
| DELETE | `/slurmdb/{data_parser}/wckey/{id}` | 刪除 WCKey |

### 實例

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/instances/` | 取得實例清單 |
| GET | `/slurmdb/{data_parser}/instance/` | 取得實例資訊 |

### 系統

| 方法 | 路徑 | 摘要 |
|------|------|------|
| GET | `/slurmdb/{data_parser}/ping/` | Ping 測試 |
| GET | `/slurmdb/{data_parser}/diag/` | 取得 slurmdbd 診斷資訊 |
| GET | `/slurmdb/{data_parser}/config` | 傾印所有設定 |
| POST | `/slurmdb/{data_parser}/config` | 載入所有設定 |

---

## 資料解析器版本

`{data_parser}` 路徑參數指定 API 版本：
- `v0.0.42` - Slurm 24.11
- `v0.0.43` - Slurm 25.05
- `v0.0.44` - Slurm 25.11
- `v0.0.45` - Slurm 26.05（目前版本）

---

## 驗證

### JWT 權杖驗證
```bash
curl -H "X-SLURM-USER-TOKEN: $JWT_TOKEN" \
     http://localhost:6820/slurm/v0.0.45/jobs/
```

### 本機 Unix Socket
```bash
curl --unix-socket /var/run/slurmrestd.socket \
     http://localhost/slurm/v0.0.45/jobs/
```

---

## 回應格式

所有回應遵循 OpenAPI 格式，包含：
- `meta` - 回應元資料（外掛、Slurm 版本）
- `errors` - 錯誤物件陣列
- `warnings` - 警告物件陣列
- 基於端點的資料特定欄位

範例：
```json
{
  "meta": {
    "plugin": {"type": "openapi/slurmctld", "name": "Slurm OpenAPI slurmctld"},
    "slurm": {"version": {"major": 26, "minor": 5, "micro": 0}, "release": "26.05.0"}
  },
  "errors": [],
  "warnings": [],
  "jobs": [...]
}
```

---

## 原始碼檔案

| 外掛 | 位置 |
|------|------|
| slurmctld OpenAPI | `src/slurmrestd/plugins/openapi/slurmctld/api.c` |
| slurmdbd OpenAPI | `src/slurmrestd/plugins/openapi/slurmdbd/api.c` |
| REST 驗證 | `src/slurmrestd/rest_auth.c` |
| HTTP 處理器 | `src/slurmrestd/http.c` |
