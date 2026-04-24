# Slurm TLS 憑證管理員

## TL;DR

`certmgr` 外掛介面可搭配 `tls` 外掛，為 slurmd/sackd 節點動態產生並更新已簽署的 TLS 憑證。目前提供 `certmgr/script` 外掛，允許透過自訂腳本完成節點身份驗證與憑證簽署流程。節點在取得已簽署憑證之前，不會開始處理任何 RPC 請求。

---

## 翻譯

## TLS 憑證管理員（TLS Certificate Manager）

### 概觀

`certmgr` 外掛介面可搭配 `tls` 外掛介面一起使用，為 slurmd/sackd 節點動態建立並更新已簽署的憑證（signed certificates）。

使用 certmgr 產生的已簽署憑證及對應的私密金鑰（private key），會在從 slurmctld 取回時儲存於 slurmd 的緩衝目錄（spool directory），並於 slurmd 啟動時載入。

---

### certmgr/script

`certmgr/script` 外掛允許透過腳本執行節點身份驗證與已簽署憑證產生所需的相關操作。

#### OpenSSL 範例

以下範例使用 openssl 命令列工具來產生憑證簽署請求（Certificate Signing Request，CSR）並簽署，以建立已簽署憑證。此範例**不適合用於正式環境**，僅用於說明各腳本的預期職責。

在本範例中，Slurm 進行憑證管理之前，需要在每台機器上預先準備以下事項。請注意，適用於 slurmd 的說明同樣適用於 sackd 節點。

slurmctld 需要能存取 CA 憑證（Certificate Authority certificate），且 CA 憑證/金鑰組必須由 `SlurmUser` 所擁有（**正式環境中不建議這樣做**）。關於如何產生此憑證/金鑰組的詳細說明，請參閱 [TLS](../tls.html#s2n_openssl_example) 頁面。

需要建立並設定以下腳本。每個腳本的詳細說明請參閱 [CertmgrParameters](../slurm.conf.html#OPT_CertmgrParameters)：

- `get_node_token_script`
- `generate_csr_script`
- `validate_node_script`
- `sign_csr_script`

slurmctld 需要能驗證 slurmd 的憑證簽署請求。驗證流程如下：在 slurmd 節點上透過 `get_node_token_script` 取得唯一權杖（token），再於 slurmctld 主機上透過 `validate_node_script` 加以驗證。

每個 slurmd 都需要產生一個唯一權杖。各權杖會儲存在對應的 slurmd 主機上，同時也會加入 slurmctld 主機上包含所有節點權杖的總清單中。slurmd 在執行期間會將權杖連同 CSR 一起傳送給 slurmctld，slurmctld 在產生已簽署憑證之前會先驗證此權杖。請注意，**slurmd 在載入已簽署憑證之前，不會開始處理任何 RPC 請求**。

以下是產生並儲存這些權杖的簡單範例：

```bash
# 產生 base64 編碼的 32 字元隨機權杖
base64 /dev/urandom | head -c 32 > ${NODENAME}_token.txt

# 將權杖加入權杖清單
echo "`cat ${NODENAME}_token.txt`" >> node_token_list.txt
```

節點 **n1** 在開機時需要有 `n1_token.txt`，或透過安全方式將其傳送至該節點。**slurmctld** 需要能安全存取 `node_token_list.txt`，以便 `validate_node_script` 驗證節點權杖。

`get_node_token_script`、`generate_csr_script` 和 `get_node_cert_key_script` 的路徑需要指向存在於 slurmd 節點上且可執行的腳本。

##### get_node_token_script 範例

將權杖輸出至標準輸出（stdout）。成功時回傳退出碼 0，發生錯誤時回傳非零退出碼。

```bash
#!/bin/bash

# Slurm 節點名稱以引數 $1 傳入
TOKEN_PATH=/etc/slurm/certmgr/$1_token.txt

# 確認權杖檔案是否存在
if [ ! -f $TOKEN_PATH ]
then
    echo "$BASH_SOURCE: Failed to resolve token path '$TOKEN_PATH'"
    exit 1
fi

# 將權杖輸出至 stdout
cat $TOKEN_PATH

# 以退出碼 0 表示成功
exit 0
```

##### generate_csr_script 範例

將憑證簽署請求（CSR）輸出至標準輸出。成功時回傳退出碼 0，發生錯誤時回傳非零退出碼。

```bash
#!/bin/bash

# Slurm 節點名稱以引數 $1 傳入
NODE_PRIVATE_KEY=/etc/slurm/certmgr/$1_private_key.pem

# 確認節點私密金鑰檔案是否存在
if [ ! -f $NODE_PRIVATE_KEY ]
then
    echo "$BASH_SOURCE: Failed to resolve node private key path '$NODE_PRIVATE_KEY'"
    exit 1
fi

# 使用節點私密金鑰產生 CSR 並輸出至 stdout
openssl req -new -key $NODE_PRIVATE_KEY \
    -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=$1"

# 檢查 openssl 的退出碼
if [ $? -ne 0 ]
then
    echo "$BASH_SOURCE: Failed to generate CSR"
    exit 1
fi

# 以退出碼 0 表示成功
exit 0
```

##### get_node_cert_key_script 範例

將用於產生 CSR 的私密金鑰輸出至標準輸出。成功時回傳退出碼 0，發生錯誤時回傳非零退出碼。

```bash
#!/bin/bash

# Slurm 節點名稱以引數 $1 傳入
NODE_PRIVATE_KEY=/etc/slurm/certmgr/$1_cert_key.pem

# 確認節點私密金鑰檔案是否存在
if [ ! -f $NODE_PRIVATE_KEY ]
then
    echo "$BASH_SOURCE: Failed to resolve node private key path '$NODE_PRIVATE_KEY'"
    exit 1
fi

cat $NODE_PRIVATE_KEY

# 以退出碼 0 表示成功
exit 0
```

`validate_node_script` 和 `sign_csr_script` 的路徑需要指向存在於 **slurmctld** 上且可執行的腳本。

##### validate_node_script 範例

節點權杖有效時回傳退出碼 0，節點權杖無效或發生其他錯誤時回傳非零退出碼。

```bash
#!/bin/bash

# 節點的唯一權杖以引數 $1 傳入
NODE_TOKEN=$1
NODE_TOKEN_LIST_FILE=/etc/slurm/certmgr/node_token_list.txt

# 確認節點權杖清單檔案是否存在
if [ ! -f $NODE_TOKEN_LIST ]
then
    echo "$BASH_SOURCE: Failed to resolve node token list path '$NODE_TOKEN_LIST'"
    exit 1
fi

# 確認唯一節點權杖是否存在於權杖清單檔案中
grep $1 $NODE_TOKEN_LIST_FILE

# 檢查 grep 的退出碼以確認是否找到權杖
if [ $? -ne 0 ]
then
    echo "$BASH_SOURCE: Failed to validate token '$NODE_TOKEN'"
    exit 1
fi

# 以退出碼 0 表示成功（節點權杖有效）
exit 0
```

##### sign_csr_script 範例

將已簽署憑證輸出至標準輸出。成功時回傳退出碼 0，發生錯誤時回傳非零退出碼。

```bash
#!/bin/bash

# 憑證簽署請求以引數 $1 傳入
CSR=$1
CA_CERT=/etc/slurm/certmgr/root_cert.pem
CA_KEY=/etc/slurm/certmgr/root_key.pem

# 確認 CA 憑證檔案是否存在
if [ ! -f $CA_CERT ]
then
    echo "$BASH_SOURCE: Failed to resolve CA certificate path '$CA_CERT'"
    exit 1
fi

# 確認 CA 私密金鑰的權限
if [ `stat -c "%a" $CA_KEY` -ne $KEY_PERMISSIONS ]
then
    echo "$BASH_SOURCE: Bad permissions for CA private key at '$CA_KEY'. Permissions should be $KEY_PERMISSIONS"
    exit 1
fi

# 使用 CA 憑證和 CA 私密金鑰簽署 CSR，並將已簽署憑證輸出至 stdout
openssl x509 -req -CA $CA_CERT -CAkey $CA_KEY 2>/dev/null <<< $CSR

# 檢查 openssl 的退出碼
if [ $? -ne 0 ]
then
    echo "$BASH_SOURCE: Failed to generate signed certificate"
    exit 1
fi

# 以退出碼 0 表示成功
exit 0
```

---

### 驗證設定是否正確

若所有設定都正確，在啟用 [DebugFlags=TLS](../slurm.conf.html#OPT_TLS) 的情況下，以下訊息應會出現在 slurmd 和 slurmctld 的日誌中。

**slurmd 日誌：**

```
slurmd: certmgr/script: certmgr_p_get_node_token: TLS: Successfully retrieved unique node token
slurmd: certmgr/script: certmgr_p_generate_csr: TLS: Successfully generated csr:
-----BEGIN CERTIFICATE REQUEST-----
. . .
-----END CERTIFICATE REQUEST-----
```

**slurmctld 日誌：**

```
slurmctld: certmgr/script: certmgr_p_sign_csr: TLS: Successfully validated node token
slurmctld: certmgr/script: certmgr_p_sign_csr: TLS: Successfully generated signed certificate:
-----BEGIN CERTIFICATE-----
. . .
-----END CERTIFICATE-----
```

**slurmd 日誌（取得已簽署憑證後）：**

```
slurmd: TLS: Successfully got signed certificate from slurmctld:
-----BEGIN CERTIFICATE-----
. . .
-----END CERTIFICATE-----
```

也可以使用 [DebugFlags=AuditTLS](../slurm.conf.html#OPT_AuditTLS) 來顯示憑證更新的較精簡日誌。

---

### 相關文件

- [TLS 設定](../tls.html) — TLS 外掛介面與 OpenSSL 憑證產生說明
- [slurm.conf 的 CertmgrParameters](../slurm.conf.html#OPT_CertmgrParameters) — certmgr 各腳本路徑的詳細設定參數
