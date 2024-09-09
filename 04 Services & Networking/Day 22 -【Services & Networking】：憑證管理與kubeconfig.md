### 今日目標

* 管理 cluster 中的憑證
  * Certificate Signing Request (CSR)
  * CSR 的核准與駁回
  * 使用「kubeadm cert」來管理 cluster 的重要憑證

* kubeconfig 的設定方式


在[昨天](https://ithelp.ithome.com.tw/articles/10348555)的內容中，我們了解到：

> 為了確保通訊安全， cluster 中的通訊主體需要有自己的 cert 和 key，而整個 cluster 會有一個 CA 來簽發這些 cert。

假如今天 cluster 中需要新增另一個使用者，他也要有自己的 cert(憑證) 和 key(私鑰)。而管理員必須取得 CA 的 cert 和 key，才能簽發證書給新的使用者。

一般情況下， CA 的 cert 和 key 都會放在 Master Node 上，有時為了安全起見會放置在另外的伺服器上保管，而管理員要簽發憑證就需要登入到這些伺服器上，取得 CA 的 key 來簽發證書。

但是憑證是會過期的，因此管理員除了簽發新憑證，還得更換過期的憑證，不難看出，如果每次簽發憑證都得登入 CA 所在的主機，過程相當繁瑣，且反覆的拿出 CA 的 key 來簽發憑證也不是很安全。

因此 k8s 提供了一些方式讓管理員能夠使用 kubectl 來簽發證書，主要分成兩個部分：

1. Certificate Signing Request (CSR)
2. CSR 的核准與駁回

### CSR (Certificate Signing Request)

假設今天有一個新使用者叫做 Bob，他需要申請一張 cert，申請過程如下：

* 首先，Bob 需要有一把 key：
```bash
openssl genrsa -out bob.key 2048
```

* 接著用這把 key 產生一個 CSR 。所謂 CSR 就是「憑證簽發請求」， Bob 會把這個 CSR 送給 CA 審核：
```bash
openssl req -new -key bob.key -out bob.csr -subj "/CN=bob"
```

* 現在，Bob一共有一把 key 和一個 CSR：
```bash
ll | grep bob
```
```text
-rw-r--r--  1 root root  883 Mar 15 06:13 bob.csr
-rw-------  1 root root 1675 Mar 15 06:11 bob.key
```

照一般情況，管理員得帶著這個 CSR 到 CA 所在的主機上，然後用 CA 的 key 來簽發 cert 給 Bob。

不過為了讓管理員用 kubectl 就能處理這些問題，我們可以將剛才產生的 CSR 做成一個「CertificateSigningRequest 物件」，製作過程如下：

* 首先，將 CSR 的內容轉成 base64:
```bash
cat bob.csr | base64 -w 0
```
```text
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1V6Q0NBVHNDQVFBd0RqRU1NQW9HQTFVRUF3d0RZbTlpTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQwpBUThBTUlJQkNnS0NBUUVBdEltNkRNSWdpN3pKSGs2UU94dFlvTkxuT1pENXhLeUJUU21kTnhCRWhEYlErNWFyCkdmZC8yYVVZYTZGMWgxT3NkNzZjek83dWUvOEw1bEpMLzdqYzVBZDJNMG1ZNXdENEdCalNCYUtHYkQ5NzI0dHoKdyszZWZHUnlXTGViQ2xnalpML0ZrSWhiN3F0cUpaV0N3cmJGRGNCWnNUUFgrN2NHTXNOeTBSNFluZWdvYUVETwpLeTZDdzEwOXFIYnpMNDlmeVZsQmQ4U1dLSklsTnM0VU8vUXRLMzhrbXN1c245OG1FS0JQcTl3V083UFRYekQ1CitXOEYyV0EyZUlPNnltSUsvREhkbk1oQ05zdzJpL2xLRjd4bTdaQVhkaGN0QjBsN2tidk9pK0RCS0tNRGNENVkKczVmd29hbzdCRWFOdCt2dTEvbHh4TmVyZ3Y1Qm9icUUvVXJoaXdJREFRQUJvQUF3RFFZSktvWklodmNOQVFFTApCUUFEZ2dFQkFEV1IxRytUNExMamt6K29JL1VrMlBRSjlRM3JvUnBIWHlkeUI5NUgrbndNb1JjMXVCZWhDNXU1CnA4eUFvcFNNRzJyU0RoeFNnellWVFhBd0FrRFRXY0U5eGFOeGp3bUFBcVREQUZzdDhrS01qVjlDcmEwZEZGa2QKTU12Q3VTY0toUnpodlB6d1dFTmRMZms0d0dSeVhoZTZhM0JrNzcrMEt2MnhNemR0S0x2U2Y4K3BTTEdxTUhPbwpGZ25uaUxKNXVvWGVYc1UwTVZqbUtQdi9vQnBsbVgrTXExbmRESkFMSTZZYVdabktjYkwyd0xJNGhCald0bnZNCkVuT2lXNUp5SHhzNFlnczNFU0VyRGlLTmhuSzY3NUlXNXlyaGxwdHlhS1pVQTY3ckJRVjhrNHBUL29BaHo0dGoKczdCZ0d1SzhpVGRkWHYwaFErUktvaGJOamNOLzJ6VT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
```
> base64 -w 0 可以讓 base64 的輸出不換行。

* 接著撰寫 CertificateSigningRequest 物件的 yaml 檔，並將剛才的 base64 輸出放入 spec.request 欄位：
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: bob
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1V6Q0NBVHNDQVFBd0RqRU1NQW9HQTFVRUF3d0RZbTlpTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQwpBUThBTUlJQkNnS0NBUUVBdEltNkRNSWdpN3pKSGs2UU94dFlvTkxuT1pENXhLeUJUU21kTnhCRWhEYlErNWFyCkdmZC8yYVVZYTZGMWgxT3NkNzZjek83dWUvOEw1bEpMLzdqYzVBZDJNMG1ZNXdENEdCalNCYUtHYkQ5NzI0dHoKdyszZWZHUnlXTGViQ2xnalpML0ZrSWhiN3F0cUpaV0N3cmJGRGNCWnNUUFgrN2NHTXNOeTBSNFluZWdvYUVETwpLeTZDdzEwOXFIYnpMNDlmeVZsQmQ4U1dLSklsTnM0VU8vUXRLMzhrbXN1c245OG1FS0JQcTl3V083UFRYekQ1CitXOEYyV0EyZUlPNnltSUsvREhkbk1oQ05zdzJpL2xLRjd4bTdaQVhkaGN0QjBsN2tidk9pK0RCS0tNRGNENVkKczVmd29hbzdCRWFOdCt2dTEvbHh4TmVyZ3Y1Qm9icUUvVXJoaXdJREFRQUJvQUF3RFFZSktvWklodmNOQVFFTApCUUFEZ2dFQkFEV1IxRytUNExMamt6K29JL1VrMlBRSjlRM3JvUnBIWHlkeUI5NUgrbndNb1JjMXVCZWhDNXU1CnA4eUFvcFNNRzJyU0RoeFNnellWVFhBd0FrRFRXY0U5eGFOeGp3bUFBcVREQUZzdDhrS01qVjlDcmEwZEZGa2QKTU12Q3VTY0toUnpodlB6d1dFTmRMZms0d0dSeVhoZTZhM0JrNzcrMEt2MnhNemR0S0x2U2Y4K3BTTEdxTUhPbwpGZ25uaUxKNXVvWGVYc1UwTVZqbUtQdi9vQnBsbVgrTXExbmRESkFMSTZZYVdabktjYkwyd0xJNGhCald0bnZNCkVuT2lXNUp5SHhzNFlnczNFU0VyRGlLTmhuSzY3NUlXNXlyaGxwdHlhS1pVQTY3ckJRVjhrNHBUL29BaHo0dGoKczdCZ0d1SzhpVGRkWHYwaFErUktvaGJOamNOLzJ6VT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

建立 CSR 物件：
```bash
kubectl create -f bob-csr.yaml
```
輸出：
```text
certificatesigningrequest.certificates.k8s.io/bob created
```

## CSR的核准與駁回

建立 CSR 物件後，管理員透過 kubectl 就可以查看目前有哪些 CSR：
```bash
kubectl get csr
```
```text
NAME        AGE     SIGNERNAME                                    REQUESTOR                  REQUESTEDDURATION   CONDITION
bob         2m40s   kubernetes.io/kube-apiserver-client           kubernetes-admin           <none>              Pending
csr-prvsb   11d     kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:1dl5mi    <none>              Approved,Issued
csr-zkwf6   11d     kubernetes.io/kube-apiserver-client-kubelet   system:node:controlplane   <none>              Approved,Issued
```

> Bob 的 CSR 目前是「Pending」狀態，表示管理員還沒有核准。

若確認無誤後，管理員就可以透過 kubectl 核准這個 CSR：
```bash
kubectl certificate approve bob
```
```text
certificatesigningrequest.certificates.k8s.io/bob approved
```
> 其實也是拿出 CA 的 key 來簽發 cert 給 Bob，只不過 k8s 已經在背後處理好了，我們只需要下 kubectl 就行了。

核准後再次查看 CSR 的狀態：
```bash
kubectl get csr bob
```
```text
NAME   AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
bob    4m51s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Approved,Issued
```

反之，如果管理員認為這個 CSR 有問題，可以透過 kubectl 駁回：
```bash
kubectl certificate deny <csr-name>
```

當 CSR 被核准後，就可以在 CSR yaml 的「status.certificate」欄位中找到簽發下來的 cert：
```bash
kubectl get csr bob  -o yaml
......
status:
  certificate: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5RENDQWR5Z0F3SUJBZ0lSQUxTT25jTGNyWVRhYkRmcVZKUnRRSEF3RFFZSktvWklodmNOQVFFTEJRQXcKRlRFVE1CRUdBMVVFQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBek1UVXdOak15TXpaYUZ3MHlOVEF6TVRVdwpOak15TXpaYU1BNHhEREFLQmdOVkJBTVRBMkp2WWpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDCkFRb0NnZ0VCQUxTSnVnekNJSXU4eVI1T2tEc2JXS0RTNXptUStjU3NnVTBwblRjUVJJUTIwUHVXcXhuM2Y5bWwKR0d1aGRZZFRySGUrbk16dTdudi9DK1pTUy8rNDNPUUhkak5KbU9jQStCZ1kwZ1dpaG13L2U5dUxjOFB0M254awpjbGkzbXdwWUkyUy94WkNJVys2cmFpV1Znc0syeFEzQVdiRXoxL3UzQmpMRGN0RWVHSjNvS0doQXppc3Vnc05kClBhaDI4eStQWDhsWlFYZkVsaWlTSlRiT0ZEdjBMU3QvSkpyTHJKL2ZKaENnVDZ2Y0ZqdXowMTh3K2ZsdkJkbGcKTm5pRHVzcGlDdnd4M1p6SVFqYk1Ob3Y1U2hlOFp1MlFGM1lYTFFkSmU1Rzd6b3Znd1NpakEzQStXTE9YOEtHcQpPd1JHamJmcjd0ZjVjY1RYcTRMK1FhRzZoUDFLNFlzQ0F3RUFBYU5HTUVRd0V3WURWUjBsQkF3d0NnWUlLd1lCCkJRVUhBd0l3REFZRFZSMFRBUUgvQkFJd0FEQWZCZ05WSFNNRUdEQVdnQlFsei9sUE5VeWlDZFQ5TEdNdmowM2MKWTdDRmtUQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUFOeG9ldTd2amt2T216K3ViWUNZMnJJbjU4cFNFaVpacQpqcEhtOFA1SjE4eXNnblJKNlNwR0Q4NWd5bzZKaUg5VzFLVmtFbDJWaFczR3ZjbzlxVDJJalpER1I1RnMxc0FLCllub1RaQ1JQOVhBeGlwV0xtZ09BYW1JMGhSbHBlUGUxTy9vTTE0UGV3WnNMaUFFVVlvdVFMT0M4Tks3TzRQLzAKRks5dnhGbmltbkY0TWNkcmtQUk9Sa056SVYrM3hmQUxqMG82TUhkb0tDb2gzWk05eG5BUTdDUEp1dE5LbEplSwppclBpdVp5NnN6MUdzcWM4S2VmeEZkQ0hFRmpJdG1hYWVYeTN2M0RLQU5QUFdDUTM4UkZ4QmxpbHd3VlJjOExaCktFbklSR3Jhb0cxbTlZUmVNZEVVRWJsbDNQbmNlSWJqdjJIRWZCYTFDT1NtVlRlQVpMWHE3QT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  conditions:
  - lastTransitionTime: "2024-03-15T06:37:36Z"
    lastUpdateTime: "2024-03-15T06:37:36Z"
    message: This CSR was approved by kubectl certificate approve.
    reason: KubectlApprove
    status: "True"
    type: Approved
```

不過 status.certificate 的內容是 base64 編碼，因此要解碼才是真正的 cert：
```bash
kubectl get csr bob -o jsonpath='{.status.certificate}'| base64 -d > bob.crt
```

這樣一來，我們就成功的幫Bob準備了 cert 和 key。

## Kubeconfig

當使用者擁有了 cert 和 key 後，如果有相關權限，就可以透過 kubectl 來操作 cluster：

```bash
kubectl --certificate=<user-cert> --key=<user-key> --certificat-authority=<ca-crt> --server=https://<api-server-ip>:6443 get pods
```

> 但是我們以前明明就直接下 kubectl get pods 就行了，哪來這麼多參數？

還記得在[Day 03](https://ithelp.ithome.com.tw/articles/10345660)建置 cluster 時，初始化 Master Node 後我們執行了以下操作：
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

這個管理員(root)的家目錄底下的 .kube/config 就是 **kubeconfig**，讓我們可以不用輸入一堆參數就能使用 kubectl。當有人下達 kubectl 指令時，kubectl 會去讀取 kubeconfig 來取得相關資訊，看看該使用者是否有權限操作 cluster。

kubeconfig 的內容大致分成三大部分：

1. **cluster** : 這裡會放 cluster 的 api-server 位置和 ca-crt 等訊息。
2. **user** : 這裡會放使用者的 cert 和 key。
3. **context** : 這裡會放 cluster 和 user 的對應關係。也就是「要使用哪個 user 操作哪個 cluster」。

這樣講看起來有點抽象，我們直接來看看預設的 kubeconfig：

```bash
cat /root/.kube/config
```
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.crt> #  base64 編碼的 ca.crt
    server: https://172.30.1.2:6443 #  api-server 的位置
  name: kubernetes # cluster的名稱
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes # 將 cluster和 user 對應起來，格式為<user.name>@<cluster.name>
current-context: kubernetes-admin@kubernetes # 目前正在使用的context
kind: Config
preferences: {}
users:
- name: kubernetes-admin # 也就是管理員
  user:
    client-certificate-data: <admin.crt> # base64 編碼的 admin.crt
    client-key-data: <admin.key> # base64 編碼的 admin.key
```

因此上面預設的 kubeconfig 就是告訴 kubectl：

  1. 有一個 cluster 叫做「kubernetes」，這個 cluster 的 api-server 位置是 https://172.30.1.2:6443

  2. 有一個 user 叫做「kubernetes-admin」，這個 user 的 cert 和 key 分別是 \<admin.crt> 和 \<admin.key>

  3. 有一個操作的對應關係為「 kubernetes-admin 操作 kubernetes」

  4. context 可以有很多個，但現在使用的是「kubernetes-admin@kubernetes」 (也就是 current-context)

因此，當我們以管理員(root)的身分執行 「kubectl get pods」 時，kubectl 會去 /root/.kube/config 中找到 current-context，然後找到 user 和 cluster 的資訊來操作 cluster。

我們來看一看較為複雜的環境：

  * 有三個 cluster，分別是 cluster1、cluster2-1 和 cluster2-2

  * 有兩個 user，分別是 user1、user2

    * user1 操作 cluster1
    * user2 操作 cluster2-1 和 cluster2-2

  * 預設讓 user2 操作 cluster2-1

那麼 kubeconfig 的「結構」會是這樣：
```yaml
apiVersion: v1
clusters:
- cluster:
    name: cluster1
- cluster:
    name: cluster2-1
- cluster:
    name: cluster2-2

users:
- name: user1
- name: user2

contexts:
- context:
  name: user1@cluster1
- context:
    name: user2@cluster2-1
- context:
    name: user2@cluster2-2

current-context: user2@cluster2-1
```

除此之外，我們還可以設定要「哪個 user 操作哪個 cluster 的 namespace」:
```yaml
contexts:
- context:
  name: user1@cluster1
  cluster: cluster1
  user: user1
  namespace: dev
```

## kubeconfig 的設定範例

我們以上面 Bob 的例子來看看如何設定 kubeconfig：

> 得先設定 Bob 的RBAC，也就是底下的 role 與 rolebinding。這部分會在後續的章節中提到，目前先理解為「給予 Bob 操作 Pods 的權限」。

* 創建一個新的 role，讓Bob可以操作 Pods：
```bash
kubectl create role new-user --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
```

* 將Bob和這個 role 做綁定：
```bash
kubectl create rolebinding new-user-binding-bob  --role=new-user --user=bob
```

接下來是設定 kubeconfig：

* 首先，將 Bob 的 `user` 訊息加入：
```bash
kubectl config set-credentials bob --client-key=bob.key --client-certificate=bob.crt --embed-certs=true
```
> 可以修改 ~/.kube/config 來加入，也可以直接用指令加入新的 user。要注意的是，這個「kubectl config」指令會**直接修改**到 ~/.kube/config 中的內容。

* 為 Bob 設定一個叫做「bob」的 context， 讓他能夠操作「kubernetes」這個 cluster：
```bash
kubectl config set-context bob --cluster=kubernetes --user=bob
```

* 將預設的 context 改為剛加入的「bob」：
```bash
kubectl config use-context bob
```

* 來測試一下:
```bash
kubectl get pods
# 成功的話會看到pods的資訊
```

不過如果換成其他 resource，就會看到錯誤訊息：
```bash
kubectl get deployments.apps
```
```text 
Error from server (Forbidden): deployments.apps is forbidden: User "bob" cannot list resource "deployments" in API group "apps" in the namespace "default"
``` 

> 因為RBAC的設定，Bob 只能操作 Pods，因此當他嘗試操作 Deployments 時就會被拒絕。

最後，切換為管理員的 context：
```bash
kubectl config use-context kubernetes-admin@kubernetes
```

---

> **Tips：權限查詢**

如過要確認某個使用者具備的權限，我們可以用「kubectl auth can-i」來測試：

```bash
kubectl auth can-i <verb> <resource> --as <user>
```

例如: 

* Bob 可以「get」 Pods 嗎？
```bash
kubectl auth can-i get pods --as bob
```
```text
yes
```

* Bob 可以「get」 Deployments 嗎？
```bash
kubectl auth can-i get deployments --as bob
```
```text
no
```

***

### kubeconfig 的基本操作彙整

一個環境中可以存在多份 kubeconfig，通常可以在 kubectl 中加入 `--kubeconfig` 來指定要使用的 kubeconfig。通常有關 kubeconfig 的操作都是透過 kubectl config 來完成的：

* 檢視目前的 kubeconfig：
```bash
kubectl config view
```

* 檢視其他的 kubeconfig：
```bash
kubectl config view --kubeconfig=<path-to-kubeconfig>
```

* 查看目前的 context：
```bash
kubectl config current-context
```

* 切換其他的 context：
```bash
kubectl config use-context <context-name>
```

* 查看 kubeconfig 定義了哪些 cluster、user 和 context：
```bash
kubectl config get-clusters
kubectl config get-users
kubectl config get-contexts
```

* 加入新的 user：
```bash
kubectl config set-credentials <user-name> --client-key=<user.key> --client-certificate=<user.crt> --embed-certs=true
```

* 加入新的 cluster：
```bash
kubectl config set-cluster <cluster-name> --server=<api-server-url> --certificate-authority=<ca.crt> --embed-certs=true
```

* 加入新的 context：
```bash
kubectl config set-context <context-name> --cluster=<cluster-name> --user=<user-name>
```

* 使用特定的 kubeconfig 來操作 kubectl：
```bash
kubectl <command> --kubeconfig=<path-to-kubeconfig>
```

相關的其他操作可以參考 kubectl config -h，或參考[官網](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)。(官網真的寫的很完整，又有範例)

### 使用 kubeadm 來管理 cluster 的重要憑證

在 kubeadm 初始化 cluster 時，會產生一些重要的憑證，例如之前介紹過的 apiserver.crt、ca.crt、etcd.crt 等等。

而這些憑證的「有效期限」相當重要，為了避免過期了沒有及時準備好換新憑證，我們可以透過 kubeadm 的「certs」指令來查看這些憑證的有效期限：
```bash
kubeadm certs check-expiration
```
```text
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jul 03, 2025 18:34 UTC   346d            ca                      no      
apiserver                  Jul 03, 2025 18:34 UTC   346d            ca                      no      
apiserver-etcd-client      Jul 03, 2025 18:34 UTC   346d            etcd-ca                 no      
apiserver-kubelet-client   Jul 03, 2025 18:34 UTC   346d            ca                      no      
controller-manager.conf    Jul 03, 2025 18:34 UTC   346d            ca                      no      
etcd-healthcheck-client    Jul 03, 2025 18:34 UTC   346d            etcd-ca                 no      
etcd-peer                  Jul 03, 2025 18:34 UTC   346d            etcd-ca                 no      
etcd-server                Jul 03, 2025 18:34 UTC   346d            etcd-ca                 no      
front-proxy-client         Jul 03, 2025 18:34 UTC   346d            front-proxy-ca          no      
scheduler.conf             Jul 03, 2025 18:34 UTC   346d            ca                      no      
super-admin.conf           Jul 03, 2025 18:34 UTC   346d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jul 01, 2034 18:34 UTC   9y              no      
etcd-ca                 Jul 01, 2034 18:34 UTC   9y              no      
front-proxy-ca          Jul 01, 2034 18:34 UTC   9y              no
```

如果發現某個憑證快要過期，可以透過 kubeadm 來更新這些憑證：
```bash
kubeadm certs renew <cert-name>
```

### 今日小結

今天示範了簽發證書給使用者，從產生金鑰、產生 CSR 到核准 CSR，最後再將 CSR 轉換成 cert。有了 cert 和 key 後，我們就可以透過 kubeconfig 來管理使用者的權限，規範使用者可以操作哪個 cluster 的哪些資源，甚至能將使用者的權限限制在某個 namespace。

到今天有關憑證的部分就告一段落了，明天我們來看看 k8s 關於網路的一個重要概念：Ingress。(順便把之前[金絲雀部署](https://ithelp.ithome.com.tw/articles/10348066)的坑填上)

----

**參考資料**

* [Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)

* [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

* [How to issue a certificate for a user](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)

* [kubectl config](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)

* [Kubernetes: Kubeconfig File](https://yuminlee2.medium.com/kubernetes-kubeconfig-file-4aabe3b04ade)