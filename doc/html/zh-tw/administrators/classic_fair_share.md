# Slurm 傳統公平分享演算法 (Classic Fairshare Algorithm)

## TL;DR

從 19.05 版本起，Fair Tree 演算法為預設，傳統公平分享演算法需設定 `PriorityFlags=NO_FAIR_TREE` 才可使用。公平分享因子 F = 2^(-U/S/d)，其中 S 是標準化份額，U 是標準化使用量（含半衰期衰減）。有效使用量計算考慮帳戶階層結構，確保同帳戶內所有用戶的總使用量符合分配比例。

---

## 翻譯

### 概觀

從 19.05 版本起，Fair Tree 演算法現為預設，傳統公平分享演算法僅在明確設定 `PriorityFlags=NO_FAIR_TREE` 時可用。

---

### 標準化份額

公平分享階層代表分配給不同專案的運算資源比例。這些分配被指派給帳戶，可以有多層分配，因為給定帳戶的分配可進一步劃分給子帳戶。

**範例**：假設機器資源分配給四個帳戶 A、B、C 和 D，帳戶 A 的份額進一步分配給子帳戶 A1 到 A4。如果有 10 個用戶在帳戶 A3 中獲得相等份額，他們每人將分配到機器的 1%。

#### 標準化份額公式

```
S = (S_user / S_siblings) *
    (S_account / S_sibling-accounts) *
    (S_parent / S_parent-siblings) * ...
```

| 變數 | 說明 |
|------|------|
| S | 用戶的標準化份額（0 到 1 之間）|
| S_user | 分配給用戶的帳戶份額數 |
| S_siblings | 分配給所有被允許使用該帳戶的用戶的總份額數 |
| S_account | 分配給帳戶的父帳戶份額數 |
| S_sibling-accounts | 分配給父帳戶所有子帳戶的總份額數 |
| S_parent | 分配給父帳戶的祖父帳戶份額數 |
| S_parent-siblings | 分配給祖父帳戶所有子帳戶的總份額數 |

---

### 標準化使用量

分配給每個作業的處理器×秒會即時追蹤。

#### 固定時間段的簡單計算

```
U_N = U_user / U_total
```

| 變數 | 說明 |
|------|------|
| U_N | 標準化使用量（0 到 1 之間）|
| U_user | 用戶在給定帳戶下的所有作業在固定時間段內消耗的處理器×秒 |
| U_total | 同一時間段內整個叢集使用的總處理器×秒數 |

#### 半衰期衰減

實際使用量跨越多個時間段。Slurm 的公平分享優先順序計算對最近的資源使用更重視，對過去的使用較不重視。

**歷史使用量公式**：
```
U_H = U_current_period +
      (D * U_last_period) + (D² * U_period-2) + ...
```

| 變數 | 說明 |
|------|------|
| U_H | 受半衰期衰減影響的歷史使用量 |
| U_current_period | 當前測量期間的使用量 |
| U_last_period | 上一測量期間的使用量 |
| D | 衰減因子（0 到 1 之間），基於 `PriorityDecayHalfLife` 設定 |

**總使用量衰減**：
```
R_H = R_current_period +
      (D * R_last_period) + (D² * R_period-2) + ...
```

**跨多時間段的標準化使用量**：
```
U = U_H / R_H
```

---

### 簡化公平分享公式

跨多時間段並受半衰期衰減影響的公平分享因子計算：

```
F = 2^(-U/S/d)
```

| 變數 | 說明 |
|------|------|
| F | 公平分享因子 |
| S | 標準化份額 |
| U | 含半衰期衰減的標準化使用量 |
| d | FairShareDampeningFactor（設定參數，預設值 1）|

#### 公平分享因子解讀

| 因子值 | 意義 |
|--------|------|
| 1 | 最高優先順序 |
| 0.5 | 使用量恰好等於分配的份額 |
| > 0.5 | 使用量少於分配的份額 |
| < 0.5 | 使用量超過分配的份額 |
| 0 | 最低優先順序 |

---

### 帳戶階層下的公平分享因子

上述方法計算的優先順序基於分配給用戶的機器比例和該用戶在特定帳戶下執行的所有作業的歷史使用量。

然而，需要另一層「公平性」來考慮從同一帳戶提取的其他用戶的使用量。這允許作業的公平分享因子受到從同一帳戶提取的其他用戶作業所交付的運算資源的影響。

**範例**：如果給定帳戶有兩個成員，且其中一個用戶在該帳戶下執行了許多作業，未執行任何作業的用戶提交的作業優先順序將受到負面影響。這確保向帳戶收取的總使用量與分配給該帳戶的機器比例相符。

---

### Slurm 公平分享公式

Slurm 公平分享公式旨在根據每個帳戶的分配和使用量為用戶提供公平排程。

**公式**：
```
F = 2^(-U_E/S)
```

差異在於使用量項是**有效使用量**：

```
U_E = U_Achild +
      ((U_Eparent - U_Achild) * S_child / S_all_siblings)
```

| 變數 | 說明 |
|------|------|
| U_E | 子用戶或子帳戶的有效使用量 |
| U_Achild | 子用戶或子帳戶的實際使用量 |
| U_Eparent | 父帳戶的有效使用量 |
| S_child | 分配給子用戶或子帳戶的份額 |
| S_all_siblings | 分配給父帳戶所有子項的份額 |

**注意**：
- 此公式僅適用於 root 以下的第二層帳戶
- 對於 root 正下方的帳戶層，其有效使用量等於實際使用量
- 計算必須從第二層帳戶開始向下進行

---

### FairShare=parent

可以使用 sacctmgr 的 `FairShare=parent` 選項在公平分享階層的某些層級停用公平分享。

對於設定 `FairShare=parent` 的用戶和帳戶，計算公平分享優先順序時將使用階層中父項的標準化份額和有效使用量值。

如果帳戶中的所有用戶都設定為 `FairShare=parent`，結果是從該帳戶提取的所有作業將獲得相同的公平分享優先順序，基於帳戶的總使用量。不會根據用戶的個人使用量增加額外的公平性。

---

### 範例

假設機器的運算資源分配給帳戶 A 和 D，分別有 40 和 60 份額：

- 帳戶 A 分為子帳戶 B（30 份額）和 C（10 份額）
- 帳戶 D 分為子帳戶 E（25 份額）和 F（35 份額）
- 用戶 1 被授權使用帳戶 B
- 用戶 2 和 3 各在帳戶 C 中獲得 1 份額
- 用戶 4 是帳戶 E 的唯一成員
- 用戶 5 是帳戶 F 的唯一成員

**標準化份額**：
- 用戶 1：0.3
- 用戶 2：0.05
- 用戶 3：0.05
- 用戶 4：0.25
- 用戶 5：0.35

**假設使用量**：
- 用戶 1 實際使用量：0.2
- 用戶 2 實際使用量：0.25
- 用戶 4 實際使用量：0.25

**帳戶有效使用量計算**：
- 帳戶 A 有效使用量 = 0.45（實際使用量）
- 帳戶 D 有效使用量 = 0.25（實際使用量）
- 帳戶 B 有效使用量 = 0.2 + ((0.45 - 0.2) * 30/40) = 0.3875
- 帳戶 C 有效使用量 = 0.25 + ((0.45 - 0.25) * 10/40) = 0.3
- 帳戶 E 有效使用量 = 0.25 + ((0.25 - 0.25) * 25/60) = 0.25
- 帳戶 F 有效使用量 = 0.0 + ((0.25 - 0.0) * 35/60) = 0.1458

**用戶有效使用量計算**：
- 用戶 1：0.2 + ((0.3875 - 0.2) * 1/1) = 0.3875
- 用戶 2：0.25 + ((0.3 - 0.25) * 1/2) = 0.275
- 用戶 3：0.0 + ((0.3 - 0.0) * 1/2) = 0.15
- 用戶 4：0.25 + ((0.25 - 0.25) * 1/1) = 0.25
- 用戶 5：0.0 + ((0.1458 - 0.0) * 1/1) = 0.1458

**公平分享因子**（F = 2^(-U_E/S)）：
- 用戶 1：2^(-0.3875/0.3) = 0.408479
- 用戶 2：2^(-0.275/0.05) = 0.022097
- 用戶 3：2^(-0.15/0.05) = 0.125000
- 用戶 4：2^(-0.25/0.25) = 0.500000
- 用戶 5：2^(-0.1458/0.35) = 0.749154

**結論**：用戶 1、2、3 是過度服務的，用戶 5 是服務不足的。即使用戶 3 尚未提交作業，其公平分享因子也受到用戶 1 和 2 執行作業的負面影響。如果所有 5 個用戶同時提交作業，用戶 5 的作業將獲得最高排程優先順序。

---

## 說明

### 傳統公平分享 vs Fair Tree

| 特性 | 傳統公平分享 | Fair Tree |
|------|--------------|-----------|
| 預設狀態 | 19.05 前預設 | 19.05+ 預設 |
| 啟用方式 | PriorityFlags=NO_FAIR_TREE | 預設 |
| 演算法 | 基於有效使用量 | 基於層級比較 |
| 複雜度 | 較複雜 | 較簡單 |

### 公平分享概念圖

```
                    Root (100%)
                    /         \
                   /           \
            帳戶 A (40%)    帳戶 D (60%)
            /       \          /      \
           /         \        /        \
      帳戶 B      帳戶 C   帳戶 E    帳戶 F
      (30%)       (10%)    (25%)     (35%)
        |         /   \      |         |
        |        /     \     |         |
      用戶1   用戶2   用戶3  用戶4     用戶5
```

---

## 實務範例

### 基本公平分享設定

**slurm.conf：**
```
PriorityType=priority/multifactor
PriorityFlags=NO_FAIR_TREE
PriorityDecayHalfLife=7-0
PriorityCalcPeriod=5
FairShareDampeningFactor=1
```

### 帳戶階層設定

```bash
# 建立帳戶階層
sacctmgr add account root
sacctmgr add account AccountA parent=root fairshare=40
sacctmgr add account AccountD parent=root fairshare=60
sacctmgr add account AccountB parent=AccountA fairshare=30
sacctmgr add account AccountC parent=AccountA fairshare=10

# 新增用戶
sacctmgr add user user1 account=AccountB fairshare=1
sacctmgr add user user2 account=AccountC fairshare=1
sacctmgr add user user3 account=AccountC fairshare=1
```

### 使用 FairShare=parent

```bash
# 讓所有用戶使用帳戶的公平分享
sacctmgr modify user where account=AccountB set fairshare=parent
```

### 檢視公平分享資訊

```bash
# 查看用戶公平分享
sshare -a

# 詳細顯示
sshare -a -l

# 查看特定帳戶
sshare -A AccountA
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 NO_FAIR_TREE | 19.05+ 需明確設定才能使用傳統公平分享 |
| 份額分配不當 | 確保階層各層份額總和合理 |
| 忽略衰減設定 | 根據站點需求調整 PriorityDecayHalfLife |
| 未考慮帳戶階層 | 正確設計帳戶階層以反映組織結構 |

### 調校建議

1. **選擇適當的半衰期**：
   - 短（數天）：快速反應近期使用
   - 長（數週/月）：較穩定的優先順序

2. **FairShareDampeningFactor**：
   - 較大值減弱公平分享的影響
   - 預設值 1 提供標準行為

3. **監控使用量**：
   ```bash
   sshare -a -l | sort -k5 -n
   ```

---

## 快速參考

### slurm.conf 設定

```
# 啟用傳統公平分享（19.05+）
PriorityType=priority/multifactor
PriorityFlags=NO_FAIR_TREE

# 衰減設定
PriorityDecayHalfLife=7-0    # 7 天半衰期
PriorityCalcPeriod=5          # 每 5 分鐘計算

# 阻尼因子
FairShareDampeningFactor=1
```

### 公式摘要

| 公式 | 用途 |
|------|------|
| S = Π(S_level/S_siblings) | 標準化份額 |
| U = U_H / R_H | 標準化使用量 |
| U_E = U_A + (U_Ep - U_A) * S_c/S_s | 有效使用量 |
| F = 2^(-U_E/S) | 公平分享因子 |

### sshare 欄位說明

| 欄位 | 說明 |
|------|------|
| Account | 帳戶名稱 |
| User | 用戶名稱 |
| RawShares | 原始份額 |
| NormShares | 標準化份額 |
| RawUsage | 原始使用量 |
| NormUsage | 標準化使用量 |
| EffectvUsage | 有效使用量 |
| FairShare | 公平分享因子 |

### 相關文件

- [Fair Tree 演算法](fair_tree.md) - 新預設演算法
- [多因子優先順序](priority_multifactor.md) - 優先順序設定
- [QoS](qos.md) - 服務品質設定
- [計費](accounting.md) - 計費系統
