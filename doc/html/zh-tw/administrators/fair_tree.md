# Slurm Fair Tree 公平分享演算法

## TL;DR

Fair Tree 是 Slurm 19.05+ 的預設公平分享演算法。它確保高優先順序帳戶的所有用戶都比低優先順序帳戶的所有用戶獲得更高的公平分享因子。演算法使用層級公平分享值 (LF = S/U) 對關聯樹進行深度優先遍歷排序，用戶排名除以總用戶關聯數即為最終公平分享因子。

---

## 翻譯

### 介紹

Fair Tree 對用戶進行優先順序排序，使得如果帳戶 A 和 B 是兄弟且 A 的公平分享因子高於 B，則 A 的所有子項都將擁有比 B 的所有子項更高的公平分享因子。

**主要優點**：

| 優點 | 說明 |
|------|------|
| 帳戶優先順序繼承 | 高優先順序帳戶的所有用戶都比低優先順序帳戶的所有用戶獲得更高的公平分享因子 |
| 精確度保證 | 用戶經過排序和排名以防止精確度損失造成的錯誤，允許並列 |
| 帳戶協調者保護 | 帳戶協調者無法意外損害其用戶相對於其他帳戶用戶的優先順序 |
| 唯一性 | 用戶極不可能因計算精確度損失而擁有完全相同的公平分享因子 |
| 即時優先順序 | 新作業會立即被指派優先順序 |

---

### 最終用戶概觀

執行 `sshare -l`（小寫 L）查看以下欄位：`FairShare`、`Level FS`。注意，如果關聯沒有使用量，Level FS 值為無限大。

**優先順序規則**：
- 如果一個帳戶的 Level FS 值高於任何其他兄弟用戶或兄弟帳戶，該帳戶的所有子項將擁有比其他帳戶子項更高的 FairShare 值
- 這在關聯樹的每一層都成立

**FairShare 值計算**：
1. 使用 Fair Tree 演算法對所有用戶進行排名（降序）
2. FairShare 值 = 用戶排名 / 總用戶關聯數
3. 最高排名的用戶獲得 1.0 的公平分享值

**診斷優先順序差異**：

如果您（UserA）的 FairShare 值低於另一個用戶（UserB），請：
1. 找到第一個共同祖先帳戶
2. 在共同祖先下一層，比較您的祖先和 UserB 祖先的 Level FS 值
3. 您的祖先具有較低的 Level FS 值

**範例**：
```
root => Acct1 => Acct12 => UserA
root => Acct1 => Acct16 => UserB
```

Acct1 是 UserA 和 UserB 的第一個共同祖先。檢查 Acct12 和 Acct16 的 Level FS 值。如果 UserB 的 FairShare 值高於 UserA，則 Acct16 的 Level FS 值高於 Acct12。

---

### 演算法

Fair Tree 演算法的運作方式：

1. 為每個關聯計算層級公平分享值（Level Fairshare），僅考慮自身和兄弟的份額及使用量
2. 邏輯上建立一個有根平面樹（rooted plane tree）
3. 按層級公平分享值排序，最高值在左邊
4. 以深度優先遍歷訪問樹
5. 在前序遍歷中發現用戶時進行排名
6. 使用排名建立用戶的最終公平分享因子

**演算法步驟**：

```
設定 rank = 用戶關聯數量
從 root 開始：
    ├── 計算子樹子項的層級公平分享
    ├── 對子項排序
    └── 以降序訪問子項
        ├── 如果是用戶：指派最終公平分享因子 = rank-- / 用戶關聯總數
        └── 如果是帳戶：下降到帳戶
```

演算法僅對樹執行單次遍歷，因為所有步驟可以合併進行。

---

### 層級公平分享計算

層級公平分享（Level Fairshare）方程式：

```
LF = S / U
```

| 變數 | 名稱 | 說明 |
|------|------|------|
| LF | Level Fairshare | 關聯的層級公平分享 |
| S | Shares Norm | 關聯份額標準化為自身和兄弟份額：`S = Sraw_self / Sraw_self+siblings` |
| U | Effective Usage | 關聯使用量標準化為帳戶使用量：`U = Uraw_self / Uraw_self+siblings` |

**值範圍**：
| 項目 | 範圍 | 意義 |
|------|------|------|
| S | 0.0 ~ 1.0 | 標準化份額 |
| U | 0.0 ~ 1.0 | 標準化使用量 |
| LF | 0.0 ~ ∞ | 層級公平分享 |
| LF > 1.0 | | 服務不足（under-served）|
| LF < 1.0 | | 服務過度（over-served）|

---

### 並列處理

| 情況 | 處理方式 |
|------|----------|
| 兄弟用戶相同 Level Fairshare | 獲得相同排名 |
| 用戶與兄弟帳戶相同 Level Fairshare | 獲得該帳戶最高排名用戶的相同排名 |
| 兄弟帳戶相同 Level Fairshare | 在下降前合併其子項清單 |

---

### sshare 命令

使用 `-l`（long）參數時，sshare 會顯示 `Level FS` 值：

```bash
sshare -l
```

此欄位顯示每個關聯的值，讓用戶可以看到每層公平分享計算的結果。

**注意**：Fair Tree 不使用 Norm Usage，但仍會顯示。

---

### 設定

以下 slurm.conf 參數用於設定 Fair Tree 演算法：

| 參數 | 說明 |
|------|------|
| `PriorityType` | 設為 `priority/multifactor` |
| `PriorityCalcPeriod` | 作業半衰期衰減和 Fair Tree 計算執行的頻率（分鐘）|

**基本設定**：
```
# slurm.conf
PriorityType=priority/multifactor
PriorityCalcPeriod=5
```

---

### 重要注意事項

1. **PriorityWeightFairshare 調整**：
   - Fair Tree 對所有用戶（活躍與否）進行排名
   - 管理員必須仔細考慮如何在 priority/multifactor 外掛程式中套用其他優先順序權重
   - `PriorityWeightFairshare` 可設為比平常小得多的值
   - 可能低至用戶關聯數的 1-2 倍

2. **需要計費資料庫**：
   - Fair Tree 需要 Slurm 計費資料庫提供使用量資訊和分配的份額值

3. **scontrol reconfigure 不會立即觸發**：
   - 即使從不同演算法切換，也不會立即執行 Fair Tree 演算法
   - 您可能需要等到 `PriorityCalcPeriod` 定義的下一次迭代

---

## 說明

### Fair Tree vs 傳統公平分享

| 特性 | Fair Tree | 傳統公平分享 |
|------|-----------|--------------|
| 預設狀態 | 19.05+ 預設 | 需設定 NO_FAIR_TREE |
| 演算法 | 層級排名 | 有效使用量 |
| 帳戶隔離 | 完全隔離 | 部分影響 |
| 計算複雜度 | 較簡單 | 較複雜 |
| 精確度問題 | 使用排名避免 | 可能有 |

### Fair Tree 概念圖

```
                    Root
                   /    \
                  /      \
            帳戶 A        帳戶 B
           LF=2.0        LF=0.5
           /    \          |
          /      \         |
     帳戶 A1   帳戶 A2    用戶 3
      LF=1.5   LF=0.8     (rank 4)
        |        |
        |        |
      用戶 1   用戶 2
     (rank 1) (rank 2-3)

排序結果（降序）：
用戶 1 > 用戶 2 > 用戶 3

FairShare 值：
用戶 1: 1.00 (3/3)
用戶 2: 0.67 (2/3)
用戶 3: 0.33 (1/3)
```

### Level FS 計算範例

假設帳戶 A 有兩個子帳戶 A1 和 A2：

| 帳戶 | 原始份額 | 原始使用量 |
|------|----------|------------|
| A1 | 60 | 30% |
| A2 | 40 | 70% |

**計算**：
```
A1: S = 60/(60+40) = 0.6
    U = 0.3/(0.3+0.7) = 0.3
    LF = 0.6/0.3 = 2.0 (服務不足)

A2: S = 40/(60+40) = 0.4
    U = 0.7/(0.3+0.7) = 0.7
    LF = 0.4/0.7 = 0.57 (服務過度)
```

A1 的所有子用戶將排名高於 A2 的所有子用戶。

---

## 實務範例

### 基本設定

```
# slurm.conf - Fair Tree 設定（預設）
PriorityType=priority/multifactor
PriorityCalcPeriod=5
PriorityWeightFairshare=10000

# 如需禁用 Fair Tree 使用傳統演算法
# PriorityFlags=NO_FAIR_TREE
```

### 帳戶階層設定

```bash
# 建立帳戶階層
sacctmgr add account research fairshare=60
sacctmgr add account teaching fairshare=40

# research 帳戶子項
sacctmgr add account physics parent=research fairshare=50
sacctmgr add account chemistry parent=research fairshare=50

# 新增用戶
sacctmgr add user alice account=physics fairshare=1
sacctmgr add user bob account=physics fairshare=1
sacctmgr add user carol account=chemistry fairshare=1
sacctmgr add user david account=teaching fairshare=1
```

### 檢視 Fair Tree 結果

```bash
# 查看完整公平分享資訊
sshare -l

# 查看特定帳戶
sshare -l -A research

# 查看用戶排名
sshare -l | sort -k6 -rn
```

### 診斷優先順序問題

```bash
# 比較兩個用戶的優先順序
sshare -l -u alice,bob

# 查看關聯樹結構
sacctmgr show associations tree

# 驗證設定
scontrol show config | grep -i priority
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 認為 Fair Tree 與傳統演算法相同 | 理解帳戶隔離的概念 |
| PriorityWeightFairshare 過高 | 根據用戶數調整，可能設為較小值 |
| 未設定計費資料庫 | Fair Tree 需要 Slurm 計費資料庫 |
| 期望 reconfig 立即生效 | 等待 PriorityCalcPeriod |
| 忽略 Level FS 值 | 使用 sshare -l 診斷問題 |

### 調校建議

1. **PriorityWeightFairshare 設定**：
   - 用戶數較少：可設為 10000-100000
   - 用戶數很多：考慮降低到 1000-10000
   - 若公平分享是主要因素：適當提高

2. **PriorityCalcPeriod 設定**：
   - 預設 5 分鐘適合大多數環境
   - 高負載叢集：可考慮 10-15 分鐘
   - 需要快速反應：可降至 1-2 分鐘

3. **監控公平分享**：
   ```bash
   # 定期檢查
   sshare -l | head -20

   # 檢查是否有異常值
   sshare -l | awk '$6 > 10 || $6 < 0.1'
   ```

---

## 快速參考

### slurm.conf 設定

```
# Fair Tree（預設，無需特別設定）
PriorityType=priority/multifactor
PriorityCalcPeriod=5

# 如需使用傳統公平分享
# PriorityFlags=NO_FAIR_TREE
```

### 層級公平分享公式

```
LF = S / U

其中：
S = Sraw_self / Sraw_self+siblings  (標準化份額)
U = Uraw_self / Uraw_self+siblings  (標準化使用量)

結果解讀：
LF > 1.0 → 服務不足（應獲得更高優先順序）
LF = 1.0 → 恰好公平
LF < 1.0 → 服務過度（應獲得較低優先順序）
LF = ∞   → 無使用量（最高優先順序）
```

### sshare 欄位說明

| 欄位 | 說明 |
|------|------|
| Account | 帳戶名稱 |
| User | 用戶名稱 |
| RawShares | 原始份額 |
| NormShares | 標準化份額（S）|
| RawUsage | 原始使用量 |
| EffectvUsage | 有效使用量（U）|
| FairShare | 最終公平分享因子（排名/總數）|
| Level FS | 層級公平分享值（LF = S/U）|

### 常用命令

| 命令 | 功能 |
|------|------|
| `sshare -l` | 顯示層級公平分享資訊 |
| `sshare -l -u user1` | 查看特定用戶 |
| `sshare -l -A account1` | 查看特定帳戶 |
| `sacctmgr show assoc tree` | 顯示關聯樹 |
| `scontrol show config \| grep Priority` | 檢查優先順序設定 |

### 相關文件

- [傳統公平分享演算法](classic_fair_share.md) - NO_FAIR_TREE 模式
- [多因子優先順序](priority_multifactor.md) - 優先順序設定
- [計費](accounting.md) - 計費系統
- [QoS](qos.md) - 服務品質設定
