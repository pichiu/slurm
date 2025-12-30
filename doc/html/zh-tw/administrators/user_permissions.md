# Slurm 使用者權限 (User Permissions)

## TL;DR

Slurm 支援三種特殊使用者權限：**Operator**（操作員）可管理資料庫物件和預約；**Admin**（管理員）具有 Operator 權限加上如同 SlurmUser/root 的 slurmctld 控制權限；**Coordinator**（協調員）可在其負責的帳戶下新增使用者和子帳戶。透過 `AdminLevel` 和 sacctmgr 設定。

---

## 翻譯

### 概觀

Slurm 支援多種特殊使用者權限，如下所述。

---

### Operator（操作員）

**權限**：
- 新增、修改和移除任何資料庫物件（使用者、帳戶等）
- 新增其他操作員

**在 SlurmDBD 服務的叢集上**：
- 檢視被 PrivateData 旗標阻擋對一般使用者的資訊
- 建立/修改/刪除預約

**設定方式**：

透過使用者資料庫記錄中的 `AdminLevel` 選項設定。

```bash
sacctmgr modify user <username> set adminlevel=operator
```

詳細設定資訊請參閱[計費與資源限制](accounting.md)。

---

### Admin（管理員）

**權限**：
- 具有與資料庫中操作員相同層級的權限
- 可以如同 SlurmUser 或 root 一樣修改服務的 slurmctld 上的任何內容

**設定方式**：

透過使用者資料庫記錄中的 `AdminLevel` 選項設定。

```bash
sacctmgr modify user <username> set adminlevel=admin
```

詳細設定資訊請參閱[計費與資源限制](accounting.md)。

---

### Coordinator（協調員）

**權限**：
- 特殊的特權使用者，通常是帳戶管理員
- 可以在其協調的帳戶下新增使用者或子帳戶
- 可以變更帳戶和使用者關聯的限制
- 可以取消、重新排隊或重新指派其管轄範圍內作業的帳戶

**注意**：協調員不能將作業限制提高到超過父帳戶的限制。

**設定方式**：

透過 Slurm 資料庫中的表格設定，定義使用者及其可作為協調員的帳戶。

```bash
sacctmgr add coordinator account=<account> names=<username>
```

詳細設定資訊請參閱 [sacctmgr](sacctmgr.html) man page。

---

## 說明

### 權限層級對照

| 權限 | 資料庫管理 | slurmctld 控制 | 帳戶範圍 |
|------|-----------|----------------|----------|
| 一般使用者 | 無 | 無 | 無 |
| Coordinator | 有限（管轄帳戶內）| 有限 | 指定帳戶 |
| Operator | 完整 | 有限 | 全域 |
| Admin | 完整 | 完整 | 全域 |

### 權限繼承關係

```
Admin
  │
  └──▶ Operator（資料庫權限）
        │
        └──▶ Coordinator（帳戶範圍內）
              │
              └──▶ 一般使用者
```

---

## 實務範例

### 設定操作員

```bash
# 將使用者設為操作員
sacctmgr modify user john set adminlevel=operator

# 驗證
sacctmgr show user john format=user,adminlevel
```

### 設定管理員

```bash
# 將使用者設為管理員
sacctmgr modify user admin_user set adminlevel=admin

# 驗證
sacctmgr show user admin_user format=user,adminlevel
```

### 設定協調員

```bash
# 將使用者設為帳戶協調員
sacctmgr add coordinator account=research names=pi_user

# 查看帳戶的協調員
sacctmgr show account research format=account,coordinators

# 移除協調員
sacctmgr remove coordinator account=research names=pi_user
```

### 降級權限

```bash
# 移除管理員/操作員權限
sacctmgr modify user john set adminlevel=none
```

---

## 常見錯誤與建議

### 常見錯誤

| 錯誤 | 正確做法 |
|------|----------|
| 給予過多權限 | 按需分配最小權限 |
| 不了解協調員範圍 | 協調員只能管理其帳戶下的資源 |
| 未設定 SlurmDBD | Operator/Admin 需要 SlurmDBD |

### 建議

1. **最小權限原則**：
   - 優先使用 Coordinator 而非 Operator
   - 只有需要全域管理時才使用 Admin

2. **協調員設定**：
   - 為每個研究群組指定協調員
   - 允許 PI 自行管理其帳戶下的使用者

3. **稽核**：
   ```bash
   # 定期檢查特權使用者
   sacctmgr show user format=user,adminlevel where adminlevel!=none

   # 檢查協調員
   sacctmgr show account format=account,coordinators
   ```

---

## 快速參考

### AdminLevel 值

| 值 | 說明 |
|----|------|
| `none` | 一般使用者（預設）|
| `operator` | 操作員 |
| `admin` | 管理員 |

### sacctmgr 命令

| 命令 | 功能 |
|------|------|
| `sacctmgr modify user X set adminlevel=Y` | 設定使用者權限層級 |
| `sacctmgr add coordinator account=A names=U` | 新增帳戶協調員 |
| `sacctmgr remove coordinator account=A names=U` | 移除帳戶協調員 |
| `sacctmgr show user format=user,adminlevel` | 查看使用者權限 |
| `sacctmgr show account format=account,coordinators` | 查看帳戶協調員 |

### 相關文件

- [計費與資源限制](accounting.md) - AdminLevel 設定
- [資源限制](resource_limits.md) - 限制設定
- [QoS](qos.md) - 服務品質設定
