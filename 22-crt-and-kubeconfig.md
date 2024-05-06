# Certificate and Kubeconfig

在上一個章節中，我們了解到:
> 為了確保通訊安全，`cluster`中的通訊主體需要有自己的`cert`和`key`，而整個`cluster`會有一個`CA`來簽發這些`cert`。

假如今天`cluster`中需要新增另一個使用者，他也要有自己的`cert`和`key`。為了簽發憑證給他，管理員必須取得`CA`的`cert`和`key`。一般情況`CA`的`cert`和`key`都會放在`master node`上，有時為了安全起見會放置在另外的伺服器上保管，而管理員要簽發憑證就需要登入到這些伺服器上操作。

但是憑證是會過期的，因此管理員處了簽發新憑證，還得額外簽發給過期的憑證。

不難看出，如果每次簽發憑證都得登入`CA`所在的主機，過程相當繁瑣，且反覆的拿出`CA`的`key`來簽發憑證也不是很安全。因此`k8s`提供了一些方式讓管理員能夠使用`kubectl`來簽發證書，主要分成兩個部分:

1. `Certificate Signing Request` (CSR)
2. `CSR`的核准與駁回

## CSR (Certificate Signing Request)

假設今天有一個新使用者叫做Bob，他需要申請一張`cert`，申請過程如下:

* 首先，得先幫Bob產生一把`key`:
```bash
openssl genrsa -out bob.key 2048
```

* 接著用這把`key`產生一個`CSR`。所謂`CSR`就是「憑證簽發請求」，`Bob`會把這個`CSR`送給`CA`審核:
```bash
openssl req -new -key bob.key -out bob.csr -subj "/CN=bob"
```

* 現在，Bob一共有一把`key`和一個`CSR`:
```bash
$ ll | grep bob
-rw-r--r--  1 root root  883 Mar 15 06:13 bob.csr
-rw-------  1 root root 1675 Mar 15 06:11 bob.key
```

照一般情況，管理員得帶著這個`CSR`到`CA`所在的主機上，然後用`CA`的`key`來簽發這個`CSR`。不過為了讓管理員用`kubectl`就能處理這些問題，我們可以將剛才產生的`CSR`做成一個`CertificateSigningRequest`物件，製作過程如下:

* 首先，將`CSR`的內容轉成`base64`:
```bash
$ cat bob.csr | base64 -w 0
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1V6Q0NBVHNDQVFBd0RqRU1NQW9HQTFVRUF3d0RZbTlpTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQwpBUThBTUlJQkNnS0NBUUVBdEltNkRNSWdpN3pKSGs2UU94dFlvTkxuT1pENXhLeUJUU21kTnhCRWhEYlErNWFyCkdmZC8yYVVZYTZGMWgxT3NkNzZjek83dWUvOEw1bEpMLzdqYzVBZDJNMG1ZNXdENEdCalNCYUtHYkQ5NzI0dHoKdyszZWZHUnlXTGViQ2xnalpML0ZrSWhiN3F0cUpaV0N3cmJGRGNCWnNUUFgrN2NHTXNOeTBSNFluZWdvYUVETwpLeTZDdzEwOXFIYnpMNDlmeVZsQmQ4U1dLSklsTnM0VU8vUXRLMzhrbXN1c245OG1FS0JQcTl3V083UFRYekQ1CitXOEYyV0EyZUlPNnltSUsvREhkbk1oQ05zdzJpL2xLRjd4bTdaQVhkaGN0QjBsN2tidk9pK0RCS0tNRGNENVkKczVmd29hbzdCRWFOdCt2dTEvbHh4TmVyZ3Y1Qm9icUUvVXJoaXdJREFRQUJvQUF3RFFZSktvWklodmNOQVFFTApCUUFEZ2dFQkFEV1IxRytUNExMamt6K29JL1VrMlBRSjlRM3JvUnBIWHlkeUI5NUgrbndNb1JjMXVCZWhDNXU1CnA4eUFvcFNNRzJyU0RoeFNnellWVFhBd0FrRFRXY0U5eGFOeGp3bUFBcVREQUZzdDhrS01qVjlDcmEwZEZGa2QKTU12Q3VTY0toUnpodlB6d1dFTmRMZms0d0dSeVhoZTZhM0JrNzcrMEt2MnhNemR0S0x2U2Y4K3BTTEdxTUhPbwpGZ25uaUxKNXVvWGVYc1UwTVZqbUtQdi9vQnBsbVgrTXExbmRESkFMSTZZYVdabktjYkwyd0xJNGhCald0bnZNCkVuT2lXNUp5SHhzNFlnczNFU0VyRGlLTmhuSzY3NUlXNXlyaGxwdHlhS1pVQTY3ckJRVjhrNHBUL29BaHo0dGoKczdCZ0d1SzhpVGRkWHYwaFErUktvaGJOamNOLzJ6VT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
```
> base64 -w 0 可以讓`base64`的輸出不換行。

* 接著撰寫`CertificateSigningRequest`物件的`yaml`檔，並將剛才的`base64`輸出放入`request`欄位:
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

建立`CSR`物件:
```bash
$ kubectl create -f bob-csr.yaml
certificatesigningrequest.certificates.k8s.io/bob created
```

## CSR的核准與駁回

建立`CSR`物件後，管理員透過`kubectl`就可以檢查目前有哪些`CSR`:
```bash
$ kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                  REQUESTEDDURATION   CONDITION
bob         2m40s   kubernetes.io/kube-apiserver-client           kubernetes-admin           <none>              Pending
csr-prvsb   11d     kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:1dl5mi    <none>              Approved,Issued
csr-zkwf6   11d     kubernetes.io/kube-apiserver-client-kubelet   system:node:controlplane   <none>              Approved,Issued
```

> bob的`CSR`目前是`Pending`狀態，表示`CA`還沒有核准。

若確認無誤後，管理員就可以透過`kubectl`核准這個`CSR`:
```bash
$ kubectl certificate approve bob
certificatesigningrequest.certificates.k8s.io/bob approved

$ kubectl get csr bob
NAME   AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
bob    4m51s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Approved,Issued
```

反之，如果管理員認為這個`CSR`有問題，可以透過`kubectl`駁回:
```bash
$ kubectl certificate deny <csr-name>
```

當`CSR`被核准後，就可以在`yaml`的`status.certificate`欄位中找到簽發下來的`cert`:
```bash
$ kubectl get csr bob  -o yaml
...
....(省略)
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

不過`status.certificate`的內容是`base64`編碼，因此要解碼才是真正的`cert`:
```bash
kubectl get csr bob -o jsonpath='{.status.certificate}'| base64 -d > bob.crt
```

這樣一來，我們就成功的幫Bob準備了`cert`和`key`。

## Kubeconfig

當使用者擁有了`cert`和`key`後，就可以透過`kubectl`來操作`cluster`，例如:
```bash
kubectl --certificate=<user-cert> --key=<user-key> --certificat-authority=<ca-crt> --server=https://<api-server-ip>:6443 get pods
```

> 但是我們以前明明就直接下`kubectl get pods`就行了，哪來這麼多參數?

還記得在[Day 03](03.md)建置`cluster`時，初始化`master node`後我們執行了以下操作:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

這個管理員(root)的家目錄底下的`.kube/config`就是`kubeconfig`，就是它讓我們可以不用輸入一堆參數就能使用`kubectl`

`kubeconfig`的內容大致分成三大部分:

1. **cluster** : 這裡會放`cluster`的`api-server`位置和`ca-crt`等訊息。
2. **user** : 這裡會放使用者的`cert`和`key`。
3. **context** : 這裡會放`cluster`和`user`的對應關係。也就是「要使用哪個`user`操作哪個`cluster`」。

這樣講看起來有點抽象，我們直接來看看預設的`kubeconfig`:

```bash
cat /root/.kube/config
```
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.crt> # 這裡放的是base64編碼的ca.crt
    server: https://172.30.1.2:6443 # 這裡放的是api-server的位置
  name: kubernetes # cluster的名稱
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes # 將cluster和user對應起來，格式為<user.name>@<cluster.name>
current-context: kubernetes-admin@kubernetes # 目前正在使用的context
kind: Config
preferences: {}
users:
- name: kubernetes-admin # 也就是管理員
  user:
    client-certificate-data: <admin.crt> # 這裡放的是base64編碼的admin.crt
    client-key-data: <admin.key> # 這裡放的是base64編碼的admin.key
```

因此上面預設的`kubeconfig`就是告訴`kubectl`:
  1. 有一個`cluster`叫做`kubernetes`，它的`api-server`位置是https://172.30.1.2:6443
  2. 有一個`user`叫做`kubernetes-admin`
  3. 有一個操作的對應關係為「使用`kubernetes-admin`操作`kubernetes`」
  4. context可以有很多個，但現在要用的是`kubernetes-admin@kubernetes` (也就是`current-context`)

因此，當我們以管理員的身分執行`kubectl get pods`時，`kubectl`就會去`kubeconfig`中找到`current-context`，然後找到`user`和`cluster`的資訊來操作`cluster`。

我們來看一看較為複雜的環境:

  * 有三個`cluster`，分別是`cluster1`、`cluster2-1`和`cluster2-2`

  * 有兩個`user`，分別是`user1`、`user2`

  * `user1`操作`cluster1`、`user2`操作`cluster2`和`cluster3`

  * 預設讓`user2`操作`cluster2-1`

那麼`kubeconfig`的結構會是這樣:
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

另外，我們還可以設定要「哪個`user`操作哪個`cluster`的`namespace`」:
```yaml
contexts:
- context:
  name: user1@cluster1
  cluster: cluster1
  user: user1
  namespace: dev
```

### kubeconfig的相關操作

* 檢視目前的`kubeconfig`:
```bash
kubectl config view
```

* 檢視其他的`kubeconfig`:
```bash
kubectl config view --kubeconfig=<path-to-kubeconfig>
```

* 查看目前的`context`:
```bash
kubectl config current-context
```
* 使用其他的`context`:
```bash
kubectl config use-context <context-name>
```

* 使用特定的`kubeconfig`來操作`kubectl`指令:
```bash
kubectl <command> --kubeconfig=<path-to-kubeconfig>
```

相關的其他操作可以參考`kubectl config -h`，或參考[這裡](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)


## kubeconfig的設定範例

我們以上面Bob的例子來看看如何設定`kubeconfig`:

> 得先設定Bob的RBAC，這部分會在後面的章節中提到。先理解為「管理員給予Bob操作`pods`的權限」。

* 創建一個新的`role`，讓Bob可以操作`pods`:
```bash
kubectl create role new-user --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
```

* 將Bob和這個`role`做綁定:
```bash
kubectl create rolebinding new-user-binding-bob  --role=new-user --user=bob
```

接下來是設定`kubeconfig`:

* 首先，將Bob的`user`訊息加入`kubeconfig`:
```bash
kubectl config set-credentials bob --client-key=bob.key --client-certificate=bob.crt --embed-certs=true
```

* 為Bob設定`context`:
```bash
kubectl config set-context bob --cluster=kubernetes --user=bob
```

* 將預設的`context`改為Bob:
```bash
kubectl config use-context bob
```

* 來測試一下:
```bash
kubectl get pods
# 成功的話會看到pods的資訊
```

不過如果換成其他`resource`，就會看到錯誤訊息:
```bash
$ k get deployments.apps 
Error from server (Forbidden): deployments.apps is forbidden: User "bob" cannot list resource "deployments" in API group "apps" in the namespace "default"
``` 

> 因為RBAC的設定，Bob只能操作`pods`，因此當他嘗試操作`deployments`時就會被拒絕。

# Reference:

[How to issue a certificate for a user](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)

[kubectl config](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)