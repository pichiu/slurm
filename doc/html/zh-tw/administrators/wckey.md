# Slurm 工作負載特性金鑰 (WCKey) 管理

## TL;DR

WCKey (Workload Characterization Key) 是一種與帳戶正交的計費方式，適用於跨帳戶的專案追蹤。設定 `TrackWCKey=yes` 追蹤 WCKey，設定 `AccountingStorageEnforce=wckey` 強制要求有效 WCKey。使用 `--wckey=` 指定作業的 WCKey，用 `sacctmgr` 管理使用者的 WCKey。

---

## 翻譯

### 概觀

WCKey 是一種正交的計費方式，可以針對可能不相關的帳戶進行計費。這在來自不同帳戶的使用者都在同一個專案上工作時特別有用。

**使用場景**：
- 跨部門專案追蹤
- 研究計畫資源使用追蹤
- 特定專案成本分攤

---

### slurm(dbd).conf 設定

#### 強制 WCKey

在 slurm.conf 的 `AccountingStorageEnforce` 選項中包含 "WCKey" 將強制每個作業使用 WCKey。這表示只有使用有效 WCKey（先前透過 sacctmgr 新增的 WCKey）的作業才被允許執行。

```
# slurm.conf
AccountingStorageEnforce=wckey
```

#### 追蹤 WCKey

如果您希望追蹤作業的 WCKey 值，必須在 slurm.conf 和 slurmdbd.conf 檔案中都設定 `TrackWCKey` 選項。

```
# slurm.conf
TrackWCKey=yes

# slurmdbd.conf
TrackWCKey=yes
```

**注意**：如果在 `AccountingStorageEnforce` 中設定了 "WCKey"，`TrackWCKey` 會自動在 slurm.conf 中設定，但仍需要手動新增到 slurmdbd.conf 檔案中。

---

### sbatch/salloc/srun

#### 指定 WCKey

每個提交工具都有 `--wckey=` 選項來設定作業的 WCKey：

```bash
sbatch --wckey=project_a script.sh
salloc --wckey=research_2024 bash
srun --wckey=hpc_test ./app
```

#### 環境變數

也可以透過環境變數設定 WCKey：
- `SBATCH_WCKEY`
- `SALLOC_WCKEY`
- `SLURM_WCKEY`

```bash
export SLURM_WCKEY=my_project
sbatch script.sh
```

#### 預設 WCKey

如果沒有指定 WCKey，作業將使用使用者在該叢集的預設 WCKey（可透過 sacctmgr 設定）。

如果沒有指定 WCKey，計費記錄會附加一個 `*` 符號，表示 WCKey 未被指定。這對管理員判斷使用者是否有指定 WCKey 很有用。

---

### sacct

使用 sacct 查看 WCKey：

```bash
# 在輸出中加入 wckey 欄位
sacct --format=jobid,user,wckey,elapsed

# 篩選特定 WCKey 的作業
sacct --wckeys=project_a,project_b
```

---

### sacctmgr

sacctmgr 用於管理 WCKey。您可以新增、移除使用者的 WCKey 或列出它們。

#### 新增使用者到 WCKey

WCKey 不需要事先建立，可以直接將使用者新增到 WCKey：

```bash
sacctmgr add user da wckey=secret_project
```

#### 從 WCKey 移除使用者

```bash
sacctmgr del user da wckey=secret_project
```

#### 修改預設 WCKey

```bash
# 變更特定叢集的預設 WCKey
sacctmgr mod user da cluster=snowflake set defaultwckey=secret_project

# 變更所有叢集的預設 WCKey
sacctmgr mod user da set defaultwckey=secret_project
```

#### 列出 WCKey

```bash
# 列出使用者的 WCKey
sacctmgr show wckey user=da

# 列出所有 WCKey
sacctmgr show wckey
```

---

### sreport

WCKey 相關報表的資訊可在 [sreport manpage](sreport.html) 中找到。

```bash
# WCKey 使用報表
sreport wckey topusage
```

---

## 說明

### WCKey vs 帳戶

| 特性 | WCKey | 帳戶 |
|------|-------|------|
| 用途 | 專案追蹤 | 組織結構 |
| 階層 | 扁平 | 階層式 |
| 強制性 | 可選或強制 | 通常強制 |
| 公平分享 | 無 | 有 |

### WCKey 概念圖

```
            組織結構（帳戶）
                 │
    ┌────────────┼────────────┐
    │            │            │
 部門 A       部門 B       部門 C
    │            │            │
  user1        user2        user3
    │            │            │
    └────────────┼────────────┘
                 │
            專案追蹤（WCKey）
                 │
    ┌────────────┼────────────┐
    │            │            │
  專案 X      專案 Y      專案 Z
```

---

## 實務範例

### 基本設定

```
# slurm.conf
AccountingStorageType=accounting_storage/slurmdbd
TrackWCKey=yes

# 可選：強制 WCKey
# AccountingStorageEnforce=wckey
```

```
# slurmdbd.conf
TrackWCKey=yes
```

### 管理 WCKey

```bash
# 為多個使用者新增 WCKey
sacctmgr add user user1,user2,user3 wckey=project_alpha

# 設定預設 WCKey
sacctmgr mod user user1 set defaultwckey=project_alpha

# 查看 WCKey 指派
sacctmgr show wckey format=name,cluster,user
```

### 查詢 WCKey 報表

```bash
# 查看特定 WCKey 的使用量
sacct --wckeys=project_alpha --format=jobid,user,elapsed,cputimeraw

# 使用 sreport 產生報表
sreport wckey topusage start=2024-01-01 end=2024-12-31
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 只在 slurm.conf 設定 TrackWCKey | slurm.conf 和 slurmdbd.conf 都需要設定 |
| 強制 WCKey 但未先新增 | 先用 sacctmgr 新增使用者的 WCKey |
| 忽略 * 標記 | 檢查使用者是否正確指定 WCKey |

### 建議

1. **規劃 WCKey 命名慣例**：
   - 使用一致的命名格式（如 `project_name_year`）
   - 避免特殊字元

2. **設定預設 WCKey**：
   - 為使用者設定合理的預設 WCKey
   - 避免作業因缺少 WCKey 而失敗

3. **定期檢查**：
   ```bash
   # 檢查未指定 WCKey 的作業
   sacct --format=jobid,user,wckey | grep '\*'
   ```

---

## 快速參考

### slurm.conf 設定

```
# 追蹤 WCKey
TrackWCKey=yes

# 強制要求有效 WCKey
AccountingStorageEnforce=wckey
```

### 常用命令

| 命令 | 功能 |
|------|------|
| `sbatch --wckey=X` | 指定作業 WCKey |
| `sacctmgr add user U wckey=X` | 新增使用者到 WCKey |
| `sacctmgr del user U wckey=X` | 從 WCKey 移除使用者 |
| `sacctmgr mod user U set defaultwckey=X` | 設定預設 WCKey |
| `sacctmgr show wckey` | 列出 WCKey |
| `sacct --wckeys=X` | 查詢特定 WCKey 的作業 |
| `sreport wckey topusage` | WCKey 使用報表 |

### 相關文件

- [計費](accounting.md) - 計費系統設定
- [QoS](qos.md) - 服務品質設定
- [資源限制](resource_limits.md) - 資源限制設定
