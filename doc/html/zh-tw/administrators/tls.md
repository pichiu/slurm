# Slurm 傳輸層安全性（TLS）

## TL;DR

Slurm 透過 `tls` 外掛介面支援內部叢集通訊的傳輸層安全性（TLS）加密，目前提供基於 Amazon s2n-tls 函式庫的 `tls/s2n` 外掛。啟用時，各守護程式需要共用的 CA 憑證以及各自的憑證/金鑰對；slurmd 和 sackd 可透過 `certmgr` 外掛動態取得憑證，也可使用靜態預設憑證。

---

## 翻譯

### 概觀

可透過 `tls` 外掛介面，為 Slurm 叢集的內部通訊啟用傳輸層安全性（TLS）加密。

---

### s2n

`tls/s2n` 外掛使用 Amazon 的 TLS 實作 [s2n-tls](https://github.com/aws/s2n-tls)，這是以 C99 實作的 TLS/SSL 協定，設計目標為簡潔、輕量、高效，並以安全性為優先。

#### 安裝

從 s2n-tls 的公開 GitHub 儲存庫建置：

```bash
git clone https://github.com/aws/s2n-tls.git
cd s2n-tls/
cmake . -B build/ -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=ON
cmake --build build/ -j $(nproc)
cmake --install build/
```

請注意，cmake 的 `-DBUILD_SHARED_LIBS=ON` 旗標為必要選項，用以建置供 Slurm 使用的 `libs2n.so` 共享函式庫。

如需不同建置設定的進一步說明，請參閱 s2n-tls 的建置文件。

#### 設定

若要為 Slurm 內部叢集通訊啟用 TLS，請在 `slurm.conf` 和 `slurmdbd.conf` 中將 [TLSType](slurm.conf.html#OPT_TLSType) 選項設定為使用 `tls/s2n` 外掛。

所有 Slurm 元件都必須能夠存取共同的 CA（憑證授權機構）憑證。`slurmctld`、`slurmdbd` 和 `slurmrestd` 各自需要有唯一的憑證/金鑰對，且這些憑證必須能鏈結回指定的 CA 憑證。請注意，各憑證/金鑰對只需對各自的守護程式可存取，只有 CA 憑證檔案需要對所有 Slurm 元件可存取。

`slurmd` 和 `sackd` 可使用靜態預設的憑證/金鑰對，也可選擇使用 `certmgr` 外掛介面動態取得並更新憑證/金鑰對。若未設定 `certmgr` 外掛介面，則必須提供靜態預設的憑證/金鑰對。詳情請參閱 [TLS 憑證管理](certmgr.html)頁面。

以下是預設憑證/金鑰 PEM 檔案名稱清單，這些檔案預期位於 Slurm 的預設 `etc` 目錄中。每個檔案的絕對路徑可透過 `TLSParameters` 個別設定：

- `ca_cert.pem`
- `ctld_cert.pem`
- `ctld_cert_key.pem`
- `dbd_cert.pem`
- `dbd_cert_key.pem`
- `restd_cert.pem`
- `restd_cert_key.pem`
- `sackd_cert.pem`
- `sackd_cert_key.pem`
- `slurmd_cert.pem`
- `slurmd_cert_key.pem`

在某些情況下（例如步驟 I/O、等待步驟分配等），部分客戶端命令需要建立監聽 socket 伺服器。為了讓其他 Slurm 元件能夠連線至這些監聽 socket，它們需要一份可信任的 TLS 憑證。客戶端命令透過 `certgen` 外掛介面暫時產生自我簽署憑證，並在預先建立的 TLS 連線中安全地共享這些憑證。

預設情況下，`certgen` 外掛無需額外設定，會使用 `openssl` 命令列工具產生自我簽署的憑證/金鑰對。也可透過 [CertgenParameters](slurm.conf.html#CertgenParameters) 選擇性地設定使用不同的腳本來產生此金鑰對。

設定好 `tls/s2n` 外掛與憑證後，可在守護程式的日誌中看到以下除錯訊息：

```
debug:  tls/s2n: init: tls/s2n loaded
```

請注意，當未設定 `tls/s2n` 時，日誌中將固定出現以下訊息：

```
debug:  tls/none: init: tls/none loaded
```

設定 [DebugFlags=TLS](slurm.conf.html#OPT_TLS) 後，可在日誌中看到遠端程序呼叫（RPC）連線的詳細資訊。例如，以下是 `slurmctld` 與 `slurmdbd` 之間連線的日誌範例：

```
slurmctld: tls/s2n: tls_p_create_conn: TLS: tls/s2n: cipher suite:TLS_AES_128_GCM_SHA256, {0x13,0x01}. fd:17.
slurmctld: tls/s2n: tls_p_create_conn: TLS: tls/s2n: connection successfully created. fd:17. tls mode:client
```

```
slurmdbd: tls/s2n: tls_p_create_conn: TLS: tls/s2n: cipher suite:TLS_AES_128_GCM_SHA256, {0x13,0x01}. fd:13.
slurmdbd: tls/s2n: tls_p_create_conn: TLS: tls/s2n: connection successfully created. fd:13. tls mode:server
```

#### OpenSSL 範例

以下是使用 OpenSSL 產生 `tls/s2n` 外掛所需憑證/金鑰對的範例，這些範例僅供測試用途，不適合用於正式環境。

產生自我簽署的 CA 憑證金鑰對：

```bash
openssl ecparam -out ca_key.pem -name prime256v1 -genkey
chmod 0400 ca_key.pem
openssl req -x509 -key ca_key.pem -out ca_cert.pem -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=my_slurm_ca"
chmod 0444 ca_cert.pem
```

以 CA 憑證簽發子憑證：

```bash
openssl ecparam -out ctld_key.pem -name prime256v1 -genkey
chmod 0400 ctld_key.pem # 確認金鑰的擁有者與執行守護程式的使用者一致
openssl req -new -key ctld_key.pem -out ctld_csr.pem -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=ctld"
openssl x509 -req -in ctld_csr.pem -CA ca_cert.pem -CAkey ca_key.pem -out ctld_cert.pem -sha384
chmod 0444 ctld_cert.pem
```

---

## 相關文件

- [TLS 憑證管理](certmgr.md) — 動態憑證取得與更新設定
- [slurm.conf 參數說明](../man/slurm.conf.html) — TLSType、TLSParameters、CertgenParameters 等設定
