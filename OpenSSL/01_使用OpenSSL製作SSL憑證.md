# 使用 OpenSSL 製作 SSL 憑證

建立內部服務，也時常免不了使用 ssl 加密通訊。本篇記錄產生 RootCA 及 RootCA key ，並由 RootCA 簽署各服務所使用的憑證，以及部屬信任的方式。

Root CA 是整個公鑰基礎設施 (PKI) 的「信任源頭」。它本質上是一個**自簽憑證 (Self-Signed Certificate)**，因為沒有更高的機構為它擔保，它必須「自己證明自己」。

## 製作 RootCA

製作 Root CA 的標準流程有三個步驟：

1. 產生私鑰 (Private Key)：解密文件使用。

2. 製作簽署請情 (CSR)：

3. 自簽生成 Root CA：

![](images/create_ca_flow.png)

#### 1. 製作私鑰

```bash
openssl genrsa -aes256 -out rootCA.key 4096
```

**參數說明**

- -aes256 ：選填，AES-256 加密保護。每次使用這把私鑰簽發其他憑證時，都需要輸入此密碼。*若不想每次使用都用密碼，則不使用此參數。*

- -out  rootCA.key：必填，輸出私鑰檔名為 rootCA.key

- 4096：加密演算法使用 **RSA 4096**

#### 2. 建立憑證簽署請求 (CSR) 與自簽憑證

在傳統流程中，我們會先產生 `.csr` 檔再進行簽署；但對於 Root CA，Openssl 提供了一行指令直接完成 **產生 CSR + 自簽憑證**

```bash
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.crt
```

**參數說明**

- **`-x509`**：告訴 OpenSSL「不要輸出 CSR，請直接輸出自簽的 X.509 憑證」。

- **`-key rootCA.key`**：指定步驟 1 產生的私鑰。

- **`-sha256`**：使用 SHA-256 雜湊演算法（避免使用過時的 SHA-1）。

- **`-days 3650`**：憑證效期（3650 天 ≈ 10 年）。Root CA 通常具備較長的效期。

- **`-out rootCA.crt`**：輸出的 Root CA 公鑰憑證。



過程中的互動欄位填寫範例：

```
Country Name (2 letter code) [AU]: TW
State or Province Name (full name) [Some-State]: Taiwan
Locality Name (eg, city) []: Hsinchu
Organization Name (eg, company) []: MyLab
Common Name (e.g. server FQDN or YOUR name) []: app.mylab.local
```

#### 3. 驗證 Root CA 憑證內容

產出 `rootCA.crt` 後，可以透過 OpenSSL 檢查憑證內容，確認它是否確實具備 CA 的特性：

```bash
openssl x509 -in rootCA.crt -text -noout
```

點檢重點欄位：

1. **Issuer (簽發者) == Subject (持有者)**：證明這是自簽憑證。

2. **X509v3 Basic Constraints**：必須包含 `CA:TRUE`，代表這張憑證有權力再去簽發其他憑證。

## 製作服務使用的憑證 ，並由 RootCA 簽署

使用剛建立好的 Root CA 來簽發 **Server 憑證（伺服器憑證）** 時，有兩個現代 PKI 規範中**絕對不能忽略的重點**：

1. **SAN (Subject Alternative Name) 延伸套件**：現代瀏覽器（Chrome、Firefox、Edge）早已廢棄僅依賴 Common Name (CN) 的驗證機制。若沒設定 SAN，即便匯入 Root CA，瀏覽器依然會報 `NET::ERR_CERT_COMMON_NAME_INVALID` 錯誤。

2. **非 CA 屬性 (`CA:FALSE`)**：Server 憑證不能再拿去簽發其他憑證。



製作 Server CA 的標準流程有三個步驟：

![](D:\MyNote\images\create_server_ca_flow.png)

#### 1. 產生 server 私鑰

一般 Server 憑證建議採用 **RSA 2048-bit**（或 ECDSA prime256v1）。為了讓 Nginx / Apache / Spring Boot 等 Web Server 在重啟時不需要人工輸入密碼，此處私鑰通常**不設密碼**：

```bash
openssl genrsa -out server.key 2048
```



#### 2. 以 server.key 建立 CSR檔案

```bash
openssl req -new -key server.key -out server.csr
```

過程中的互動欄位填寫範例：

```
Country Name (2 letter code) [AU]: TW
State or Province Name (full name) [Some-State]: Taiwan
Locality Name (eg, city) []: Hsinchu
Organization Name (eg, company) []: MyLab
Common Name (e.g. server FQDN or YOUR name) []: app.mylab.local
```

#### 3. 用 Root CA 簽署 Server 憑證

為了符合現代瀏覽器安全規範，我們需要建立一個設定檔（例如 `server_ext.cnf`），將 **SAN** 及 **Key Usage** 帶入：

**1. 建立設定檔 `server_ext.cnf`**

```toml
basicConstraints = CA:FALSE
nsCertType = server
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = app.mylab.local
DNS.2 = *.mylab.local
IP.1  = 127.0.0.1
IP.2  = 192.168.1.100
```

(說明：可根據需求同時支援 FQDN、通配符 Wildcard 以及 IP 位址)

**2. 執行簽署指令**

使用先前產生的 `rootCA.crt` 與 `rootCA.key` 進行簽署：

```bash
openssl x509 -req -in server.csr \
  -CA rootCA.crt -CAkey rootCA.key -CAcreateserial \
  -out server.crt -days 365 -sha256 \
  -extfile server_ext.cnf
```

- **`-CAcreateserial`**：會自動建立 `rootCA.srl` 來記錄發放憑證的流水號。

- **`-extfile server_ext.cnf`**：套用我們剛設定好的 SAN 擴充屬性。

- **`-days 365`**：Server 憑證效期（建議設為 1 年以內）。

#### 驗證與測試憑證

簽署完成後，你會得到 `server.crt`。可以用 OpenSSL 來確認憑證鏈與 SAN 屬性：

**1. 驗證憑證鏈信任關係**

```bash
openssl verify -CAfile rootCA.crt server.crt
```

**正確輸出結果**：`server.crt: OK`

**2. 檢查 SAN 欄位是否成功寫入**

```bash
openssl x509 -in server.crt -text -noout | grep -A 2 "Subject Alternative Name"
```

**預期輸出**： `X509v3 Subject Alternative Name:` `DNS:app.mylab.local, DNS:*.mylab.local, IP Address:127.0.0.1, IP Address:192.168.1.100`
