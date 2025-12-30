# Slurm 深度無關公平分享因子

## TL;DR

深度無關公平分享因子是預設公平分享的變體，可增加可用優先順序範圍並改善深層和/或不規則階層中帳戶之間的公平性。公式 `F = 2^(-R)` 其中 R 是有效使用率。透過 `PriorityFlags=DEPTH_OBLIVIOUS` 和 `PriorityType=priority/multifactor` 啟用。

---

## 翻譯

### 簡介

深度無關公平分享因子是預設公平分享因子的變體，可增加可用的優先順序範圍並改善深層和/或不規則階層中帳戶之間的公平性。假設讀者熟悉 priority/multifactor 外掛程式，這裡只記錄深度無關因子的具體內容。

---

### 深度無關公平分享公式

計算帳戶公平分享因子的主要公式是：

```
F = 2^(-R)
```

其中：
- **F** = 公平分享因子
- **R** = 帳戶的有效使用率

此公式類似於原始的公平分享公式，對於樹的第一層帳戶（root 下）產生相同的結果。事實上，對於第一層帳戶，有效使用率 R 等於使用率 r，定義為：

```
r = U/S
```

其中：
- **S** = 正規化的份額
- **U** = 考慮半衰期衰減的正規化使用量

這與原始公式相同。

---

### 帳戶階層下的有效使用率

R 的通用公式更複雜一些。它涉及本地使用率 r<sub>l</sub>：

```
r_l = r / (U_all_siblings / S_all_siblings)
```

這是帳戶的使用率與其層級所有兄弟帳戶（包括自身）的總使用率之間的比率。

**範例**：假設一個帳戶的所有子帳戶總共使用了其合併份額（等於父帳戶的份額）的兩倍，但其中一個子帳戶只使用了其份額的三分之二，該子帳戶的本地使用率將是三分之一。

R 的通用公式定義為：

```
R = R_parent * r_l^k
```

其中：
- **k** 在 0 和 1 之間變化，決定帳戶的有效使用率受其祖先使用率影響的程度

---

### k 值的計算

為了理解 k 的公式，首先對 R 的公式做一些觀察：

**當 k = 1 時**：
- 上述公式給出 R = R<sub>parent</sub> * r<sub>l</sub>
- 對於第二層帳戶，代入 r<sub>l</sub> 的公式，得到 R = r * U<sub>parent</sub>/U<sub>all_siblings</sub>
- 假設作業在葉帳戶提交，U<sub>parent</sub> = U<sub>all_siblings</sub>，給出 R = r
- 這意味著如果 k = 1，帳戶的公平分享因子只基於其自身的使用率

**當 k = 0 時**：
- R = R<sub>parent</sub>
- 這意味著帳戶的公平分享因子只基於其祖先的使用率

**k 的公式**：

```
k = 1/(1+(5*ln(R_parent))^2)  如果 ln(R_parent)*ln(r_l) <= 0
k = 1                          如果 ln(R_parent)*ln(r_l) >= 0
```

**公式設計原理**：

- 如果帳戶祖先的使用量符合目標，帳戶的公平分享因子主要取決於其自身的使用量
- 因此當 R<sub>parent</sub> 趨近於 1 時，k 趨近於 1
- 相反，帳戶的祖先越是低於/超過其份額使用，帳戶的公平分享因子應該透過向父帳戶的公平分享因子移動來獲得獎勵/懲罰
- 因此當 R<sub>parent</sub> 偏離 1 時，k 趨近於 0
- 但是，如果帳戶的使用不平衡比其祖先在相同方向上更大（例如祖先消耗了其份額的兩倍，子帳戶消耗了其份額的三倍），將公平分享因子移回父帳戶沒有幫助
- 因此在這種情況下 k 保持為 1

---

### 設定

以下 slurm.conf 參數用於啟用深度無關風格的公平分享因子。詳細資訊請參閱 slurm.conf(5) man page。

| 參數 | 值 |
|------|-----|
| PriorityFlags | DEPTH_OBLIVIOUS |
| PriorityType | priority/multifactor |

```
# slurm.conf
PriorityType=priority/multifactor
PriorityFlags=DEPTH_OBLIVIOUS
```

---

## 說明

### 公式視覺化

```
公平分享因子計算:

F = 2^(-R)

其中 R = R_parent * r_l^k

         ┌─────────────────────────────────────┐
         │           F 隨 R 變化               │
    1.0  │*                                    │
         │ *                                   │
         │  *                                  │
    0.5  │   **                                │
         │     ***                             │
    0.0  │        *******                      │
         └─────────────────────────────────────┘
         0         1         2         3    R

R < 1: 使用不足 → F > 0.5 (優先順序提升)
R = 1: 使用符合目標 → F = 0.5
R > 1: 使用過度 → F < 0.5 (優先順序降低)
```

### k 值函數圖

```
         ┌─────────────────────────────────────┐
         │         k 隨 R_parent 變化          │
    1.0  │                *                    │
         │              * * *                  │
         │            *     *                  │
    0.5  │          *         *                │
         │        *             *              │
    0.0  │ ****                     ****       │
         └─────────────────────────────────────┘
        0.1       0.5       1       2      10
                       R_parent

R_parent ≈ 1: k ≈ 1 (帳戶使用主導)
R_parent 偏離 1: k → 0 (祖先使用主導)
```

### 階層範例

```
                    root
                      │
        ┌─────────────┼─────────────┐
        │             │             │
    帳戶 A         帳戶 B         帳戶 C
    份額:40%       份額:40%       份額:20%
        │             │
    ┌───┴───┐     ┌───┴───┐
    │       │     │       │
   A1      A2    B1      B2
  (20%)   (20%) (20%)   (20%)

傳統公平分享: A1, A2, B1, B2 可能有相似優先順序
深度無關:    考慮父帳戶使用情況調整優先順序
```

---

## 實務範例

### 啟用深度無關公平分享

```
# slurm.conf
PriorityType=priority/multifactor
PriorityFlags=DEPTH_OBLIVIOUS

# 其他多因子參數
PriorityWeightFairshare=10000
PriorityWeightAge=1000
PriorityWeightQOS=5000
PriorityDecayHalfLife=14-0
```

### 檢視公平分享資訊

```bash
# 查看帳戶公平分享
sshare -a

# 查看特定帳戶
sshare -A account_name

# 詳細資訊
sshare -l -a
```

### 深層階層設定

```bash
# 建立多層帳戶結構
sacctmgr add account dept1 parent=root
sacctmgr add account team1 parent=dept1
sacctmgr add account project1 parent=team1
sacctmgr add user john account=project1

# 設定份額
sacctmgr modify account dept1 set fairshare=100
sacctmgr modify account team1 set fairshare=50
sacctmgr modify account project1 set fairshare=25
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 未設定 PriorityType | 必須設為 priority/multifactor |
| 與其他 PriorityFlags 衝突 | 檢查 PriorityFlags 相容性 |
| 期望立即生效 | 需要時間讓使用資料累積 |

### 建議

1. **適用場景**：
   - 深層帳戶階層（3 層以上）
   - 不規則階層結構
   - 需要更精細的公平分享控制

2. **監控**：
   ```bash
   # 定期檢查公平分享分佈
   sshare -a | awk '{print $1, $NF}'

   # 檢查優先順序
   sprio -l
   ```

3. **調校**：
   - 觀察優先順序分佈
   - 必要時調整 PriorityWeightFairshare
   - 考慮 PriorityDecayHalfLife 的影響

---

## 快速參考

### slurm.conf 設定

```
PriorityType=priority/multifactor
PriorityFlags=DEPTH_OBLIVIOUS
```

### 公式摘要

| 公式 | 說明 |
|------|------|
| F = 2^(-R) | 公平分享因子 |
| r = U/S | 基本使用率 |
| r_l = r / (U_siblings/S_siblings) | 本地使用率 |
| R = R_parent * r_l^k | 有效使用率 |

### k 值規則

| 條件 | k 值 |
|------|------|
| ln(R_parent)*ln(r_l) <= 0 | 1/(1+(5*ln(R_parent))^2) |
| ln(R_parent)*ln(r_l) >= 0 | 1 |

### 相關文件

- [多因子優先順序](priority_multifactor.md) - 多因子優先順序設定
- [傳統公平分享](classic_fair_share.md) - 傳統公平分享演算法
- [Fair Tree](fair_tree.md) - Fair Tree 演算法
- [計費](accounting.md) - 計費與份額設定

