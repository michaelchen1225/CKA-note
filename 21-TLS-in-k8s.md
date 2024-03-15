# SSL/TLS in Kubernetes

在前一個章節中，我們在備份`etcd`時所使用的指令中，我們看到了幾個關鍵詞作為參數使用: `--key`、`--cert`、`--cacert`，而這都和今天的主題`SSL/TLS`有關。

## What's SSL/TLS?

* `SSL` (Secure Sockets Layer) : 一種通訊協定，用來保護網路通訊的安全性與隱私性。

* `TLS` (Transport Layer Security) : 新版的`SSL`，不過術語上有時還是以`SSL`來稱呼。

> 反正，SSL/TLS都是用來保護通訊安全與隱私的，只是版本新舊的差異而已，術語上兩者通用。

## SSL/TLS 憑證

思考一下: 如果我今天正在登入我的銀行網站www.myBank.com，有可能會遇到甚麼問題呢?

**問題1**

用明文傳密碼，與在大街上用大聲公廣播自己的密碼沒什麼兩樣。因此，我們常採用「**非對稱式加密**」來保護密碼的傳輸。

非對稱加密的特性是: 通訊雙方各自有一對金鑰，一個是`public key`，另一個是`private key`。兩個金鑰皆可拿來加密，但解密時需另一隻金鑰才能解密。(不能拿同一支key加密再解密)

> 舉例而言，拿public key加密只能用private key解密，反之亦然。

如果以上面登入銀行帳號的情境為例，非對稱加密的傳輸流程如下:

  1. 我和銀行網站互相交換`public key`(雙方各持有對方的公鑰)
  2. 我用銀行的`public key`加密我的密碼，再用我的`private key`「簽名」，然後傳給銀行。
  3. 銀行收到後，用我的`public key`驗證我的簽名，然後用銀行自己的`public key`解密我的密碼，完成登入。

> 在這樣的傳輸過程中，假設有駭客可以截獲所有的封包，但是他只能拿到「我們雙方的`public key`」和「加密的密碼」，由於缺少`private key`，因此他什麼也解不開。
 
**問題2**

山不轉人轉，那麼駭客拿不到密碼時，會不會直接做一個釣魚網站，騙我說他是真正的myBank.com呢?

對銀行網站來說，這個問題的重點是: 如何向使用者證明「我就是myBank.com」。

這時候，「**憑證**」(certificate) 就派上用場了。銀行可以向權威的憑證簽發機構，也就是所謂的**CA**(Certificate Authority) 申請憑證，`CA`會對銀行的身份進行驗證，然後使用`CA`的`private key`簽發一張具有公信力的憑證。

憑證的內容包括:
  * 網域名稱
  * 憑證簽發機構
  * 憑證簽發機構的數位簽章
  * 簽發日期
  * 到期日期
  * 公開金鑰
  * SSL/TLS 版本

有了這張憑證，銀行就可以向使用者證明「我就是myBank.com」，而使用者則可以透過`CA`的`public key`(通常在瀏覽器裡)來驗證這張憑證是不是真的由`CA`所簽發。

同樣，銀行也可以要求使用者出示憑證，以確保對方是「真正的」使用者。 

綜合以上，我們可以發現在`TLS`的通訊過程中，所有的角色(包括`CA`本身)都需要有一對`public key`和`private key`，以及一張證明自己的「憑證」。而這整套關於`public key`、`private key`和「憑證」的通訊機制，就是所謂的`PKI`(Public Key Infrastructure)，因此在`cluster`中有關憑證或鑰匙的檔案，通常都會放在`/etc/kubernetes/pki`目錄下。

> 補充: 
> 1. 所有人都可以像CA一樣簽發證書給別人，只是「公信力」的區別而已。
> 2. 更詳細的SSL/TLS運作原理說明，可以參考[這篇文章](https://www.kawabangga.com/posts/5330)。

# TLS in kubernetes

在`cluster`中，充滿各式各樣的「通訊」: 管理者透過`kubectl`與`kube-apiserver`溝通、`kube-apiserver`與`etcd`溝通、`kube-apiserver`與`kubelet`溝通、`kubelet`與`container runtime`溝通等等。這些通訊通通需要考慮安全性及隱私性的問題，因此絕大多數都使用了`TLS`，簡單來說:
> TLS is everwhere

既然如此，`cluster`中的元件都會有自己的`cert`和`key`，而整個`cluster`也會有自己的`CA`。

而憑證又依角色的不同分為兩種: `client`和`server`。例如:
- 管理員使用`kubectl`時，就是`client`，而`kube-apiserver`就是`server`

- 當`kube-apiserver`要連接`etcd`時，`kube-apiserver`就是`client`，而`etcd`就是`server`。

整體的`PKI`架構如下:

![PKI](21-k8s-tls.png)


在前面提到的`public key`，通常檔名的結為會是:
  * **.crt**
  * **.pem**

而通常`private key`的檔名結尾會是:
  * **.key**
  * **-key.pem**

> 通常有`key`的就是private key。

在經過了上面的介紹後，我們再回頭來看一次`etcdctl`的備份指令:
```bash
etcdctl --endpoints=127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

由於`etcd`的資料相當重要，因此使用`etcdctl`與`etcd`進行*client-server*通訊時，就必須透過`TLS`來保護通訊的安全性，因此我們需要提供:

  * **--cacert**: `etcd server`的`CA`憑證
  * **--cert**: `client`端的「憑證」
  * **--key**: `client`的「私鑰」

所以當`client`向`etcd`發送請求時，需要使用`client`的「私鑰」來在請求上「簽名」。`etcd`收到請求後，會使用`client`提供的的憑證(其中包含`client`的公鑰)來驗證請求上的「簽名」，以此確認請求的來源。

## 檢視憑證內容

前面講了這麼久的憑證，我們實際來看一下`kube-apiserver`的憑證到底長什麼樣子。

  * 先看看`kube-apiserver`的`yaml`檔:
```bash
 cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep tls
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt # <--- 憑證
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

 * 如果我們用常規的方式去看，會發先無法直接看到憑證的內容:
```bash
cat /etc/kubernetes/pki/apiserver.crt
```
```text
-----BEGIN CERTIFICATE-----
MIIDjDCCAnSgAwIBAgIIZvmWk5Me+YEwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0yNDAzMTMxMTQ5MzdaFw0yNTAzMTMxMTU0MzdaMBkx
(省略)
-----END CERTIFICATE-----

```

  * 因此這裡我們使用`openssl`來檢視憑證的內容:
```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```
> * **-text**: 用文本方式顯示，我們才看得懂
> * **-noout**: 不輸出憑證的編碼版本。當你只對憑證的文本信息感興趣時蠻有用的

```bash
# 憑證內容
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 7420127421642242433 (0x66f99693931ef981)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes # 簽發者，也就是CA
        Validity
            Not Before: Mar 13 11:49:37 2024 GMT
            Not After : Mar 13 11:54:37 2025 GMT # 到期日
        Subject: CN = kube-apiserver # common name，憑證擁有者
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                (省略)
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                3C:A8:9F:CD:4D:5B:DA:22:28:20:90:AD:66:19:90:3D:76:B4:7F:87
            X509v3 Subject Alternative Name:  # 主體的別名
                DNS:controlplane, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:192.26.13.6
                (省略)
```

> 需要注意憑證的到期日，過期的憑證是無法使用的!

我們再來檢視看看`etcd`的憑證:
```bash
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text -noout
```
```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1619366998176164672 (0x167925b47f47eb40)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = etcd-ca # 簽發者
        Validity
            Not Before: Mar 13 12:30:30 2024 GMT
            Not After : Mar 13 12:35:30 2025 GMT
        Subject: CN = controlplane # 憑證擁有者
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                (省略)
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                7A:7E:6A:87:01:1C:64:83:FF:B0:79:EF:C1:05:57:BF:9A:A8:DE:BF
            X509v3 Subject Alternative Name: 
                DNS:controlplane, DNS:localhost, IP Address:192.27.223.3, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1

```

可以看到，`etcd`的憑證中的`Issuer`是`etcd-ca`，與`kube-apiserver`的`Issuer`是`kubernetes`不同，所以我們可以將重要憑證的目錄整理如下:

  * **etcd相關** : `/etc/kubernetes/pki/etcd`
   
  * **kube-apiserver相關** : `/etc/kubernetes/pki`

有時候如果發現`etcd`掛掉，有可能就是誤以為所有憑證都在`/etc/kubernetes/pki`下，例如:
```bash
# 觀察kube-apiserver的log後，發現以下錯誤訊息
W0916 14:19:44.771920       1 clientconn.go:1331] [core] grpc: addrConn.createTransport failed to connect to {127.0.0.1:2379 127.0.0.1 <nil> 0 <nil>}. Err: connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority". Reconnecting...
```

結果一看，是`etcd`的`CA`指定錯了:
```bash
$ cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
    - --etcd-cafile=/etc/kubernetes/pki/ca.crt # 錯誤在這
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379

```

> 應該是`--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt`才對



# Reference

[非對稱金鑰](https://medium.com/@RiverChan/%E5%9F%BA%E7%A4%8E%E5%AF%86%E7%A2%BC%E5%AD%B8-%E5%B0%8D%E7%A8%B1%E5%BC%8F%E8%88%87%E9%9D%9E%E5%B0%8D%E7%A8%B1%E5%BC%8F%E5%8A%A0%E5%AF%86%E6%8A%80%E8%A1%93-de25fd5fa537)

[TLS/SSL名詞解釋](https://www.digicert.com/tw/what-is-ssl-tls-and-https)

[TLS/SSL運作原理](https://www.kawabangga.com/posts/5330)

[TLS/SSL運作流程](https://ithelp.ithome.com.tw/articles/10219106)

[TLS證書內容](https://aws.amazon.com/tw/what-is/ssl-certificate/)
