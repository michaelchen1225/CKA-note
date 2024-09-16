### 今日目標

---
* 了解誰需要 Service Account

* Service Account 是什麼？如何被 Pod 使用？

* 實作：使用 RBAC 設定 Service Account 的權限

* 實作：在 Pod 中使用 Service Account 來存取 k8s API

***

我們在 [Day 27](https://ithelp.ithome.com.tw/articles/10350434) 談到 RBAC，用來規範「使用者」的權限，但是在 k8s 中除了「使用者」之外，還有一種身分也有可能會和 k8s 互動，那就是「服務」。例如在 [Day 23](https://ithelp.ithome.com.tw/articles/10349141) 安裝的 「Ingress-Nginx Controller」，會和 kube-apiserver 互動來取得 service 的資訊，那該如何對這個服務做權限管理呢？

一般來說，會和 k8s 互動的對象可以分為兩種帳號：

  1. **人**：也就是一般使用者，使用 user account 做權限管理 (例如前一章的 bob)

  2. **服務**：例如一個 Pod，使用 **service Account** 做權限管理。Service account 最重要的部分就是「**token**」，這個 token 會被 Pod 用來和 kube-apiserver 互動，如果權限驗證通過才能取得資料。

> 總而言之，**任何**與 k8s 互動的對象，都需要一個帳號做身分驗證與權限管理。只要不是人，就是用 Service Account。

### Default Service Account

照上面的說明來看，你可能會說： 

> 既然任何與 k8s 互動的對象都會有權限管理，那之前建立 Pod 的時候，並沒有特別指定 Service Account，為什麼 Pod 還是能運作？

這是因為，每個 namespace 都有一個「**Default Service Account**」，當你沒有指定 Service Account 時，k8s 會自動綁定這個 Default Service Account 到 Pod上，並將 Default Service Account 的 token 掛載到 Pod 中。 

* 我們來觀察一下 Default Service Account：

```bash
kubectl get sa
```
```text               
NAME      SECRETS   AGE
default   0         15d
```

底下我們實際建立了一個新的 namespace，發現 k8s 會自動建立一個 default service account 在新的 namespace 中：

```bash
kubectl create ns new-ns
kubectl get sa -n new-ns
```
```text
NAME      SECRETS   AGE
default   0         0s
```

當我們在 new-ns 中建立一個 Pod 時，k8s 會自動綁定 default service account 到這個 Pod 上：

* 在 new-ns 中建立一個 Pod：
```bash
kubectl run nginx --image=nginx -n new-ns
```

* 觀察 Pod 的 yaml 輸出，可以看到 serviceAccountName 是 default：
```bash
kubectl get po nginx -n new-ns -o yaml | grep serviceAccountName
```
```text
    serviceAccountName: default
```

* 另外在 yaml 輸出中的 `spec.containers.volumeMounts` 中，可以看到 default service account 被掛載到了 Pod 中 的 `/var/run/secrets/kubernetes.io/serviceaccount`：

```bash
kubectl get po nginx -n new-ns -o yaml | grep -A 3 volume
```
```yaml
  volumeMounts:
  - mountPath: /var/run/secrets/kubernetes.io/serviceaccount # 掛載點
    name: kube-api-access-djh9z
    readOnly: true
```


* 我們來看一下這個掛載點都放了些什麼：
```bash
kubectl exec nginx -n new-ns -- ls /var/run/secrets/kubernetes.io/serviceaccount
```
```text
ca.crt  namespace  token
```

* 上面這些檔案都是透過「volume」掛載進來的：
```bash
kubectl get po nginx -n new-ns -o yaml | grep -A 20 volumes
```
```yaml
  volumes:
  - name: kube-api-access-djh9z
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

**補充：downwardAPI**

downwardAPI 允許容器在不經過 k8s client 端或 api-server 的情況下取得 Pod 的資訊。這些「Pod 的資訊」可以用以下兩種方式引入到容器中：

* 作為環境變數：這個在 [Day 5](https://ithelp.ithome.com.tw/articles/10345967) 有示範過。
* 用 volume 掛載到容器中，形成一個檔案，如上面的範例中的 /var/run/secrets/kubernetes.io/serviceaccount/namespace (內容為 new-ns)。

### 實作：使用 RBAC 設定 Service Account 的權限

不過為了安全起見，這個 Default Service Account 幾乎沒有任何權限，因此如果**權限許可的話**，使用者可以自己建立 Service Account 並指定給 Pod 使用，再透過 **RBAC** 來設定這個 Service Account 的權限。


我們來看一個範例：

* 在 new-ns 中建立新的 Service Account：
```bash
kubectl create sa read-pod-sa -n new-ns
```

* 建立 role，這個 role 可以讀取 new-ns namespace 中的 Pod 資訊：
```bash
kubectl create role pod-reader --verb=get,watch,list --resource=pods -n new-ns
```

* 建立 role binding，將 pod-reader 與 read-pod-sa 綁定：
```bash
kubectl create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=new-ns:read-pod-sa -n new-ns
```
> 用指令綁定 Service Account 與 Role 的時候，--serviceaccount 的格式是 `<namespace>:<service-account-name>`，要記得加上 namespace 喔。

* 驗證一下：
```bash
kubectl auth can-i get pods -n new-ns --as system:serviceaccount:new-ns:read-pod-sa 
```
```text
yes
```

### 實作：在 Pod 中使用 Service Account

要在 Pod 中使用 Service Account，需要在 yaml 中的 `spec.serviceAccountName` 中指定：

* 在 new-ns 中建立一個 Pod，並指定 service account：
```yaml
# sa-test.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sa-test
  name: sa-test
  namespace: new-ns
spec:
  containers:
  - image: nginx
    name: nginx
  serviceAccountName: read-pod-sa # 指定service account
```
```bash
kubectl apply -f sa-test.yaml
```
> **注意**：Pod 只能使用同一個 namespace 中的 Service Account。

* 建立 Pod 之後，我們開啟一個終端與 Pod 互動：Pod 資訊：
  
```bash
kubectl exec -n new-ns -it sa-test -- /bin/bash
```

* 使用 curl 測試一下，看看 sa-test 是否能透過 Service Account 的權限存取 k8s API，取得 new-ns namespace 的 Pod 列表：
```bash
CACERT='/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc.cluster.local:443/api/v1/namespaces/new-ns/pods/
```
> 在沒有 kubectl 的情況下，就可以用 curl 來存取 kube-apiserver，不過要提供身分驗證的資訊，例如 CA 憑證、token 等。

如果成功會看到 new-ns 中的 Pod 列表：
```yaml
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "2625"
  },
  "items": [
    {
      "metadata": {
        "name": "sa-test",
        "namespace": "new-ns",
        "uid": "37dcc3f0-6241-4c3d-ae03-8f1f26eebe7a",
        "resourceVersion": "2453",
        "creationTimestamp": "2024-09-15T07:25:26Z",
        "labels": {
          "run": "sa-test"
        },
.......(省略)
```

> 如果失敗，檢查看看前面步驟中的 RBAC 是否設定正確。

如果將 sa-test 的 Service Account 換成 Default Service Account，重新建立 sa-test 再次嘗試存取 k8s API，還能夠成功嗎？

* 將 sa-test 的 Service Account 換成 Default Service Account：
```bash
sed -i 's/serviceAccountName: read-pod-sa/serviceAccountName: default/' sa-test.yaml
kubectl replace --force --grace-period=0 -f sa-test.yaml
```

* 重新測試：
```bash
kubectl exec -n new-ns -it sa-test -- /bin/bash
```
```bash
CACERT='/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc.cluster.local:443/api/v1/namespaces/new-ns/pods/
```

從 kube-apiserver 返回的訊息就能看出，Default Service Account 沒有權限：
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:new-ns:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"new-ns\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

### 環境清理

> 練習後別忘了清理環境喔

* 清除 new-ns 中的所有資源：
```bash
kubectl delete all --all -n new-ns
```

* 刪除 new-ns namespace：
```bash
kubectl delete ns new-ns
```

### 今日小結

透過 Service Account 與 RBAC，我們也能針對 Pod 設定權限。

不過也不是每個人都能在 cluster 中任意建立 Service Account，畢竟還是得看使用者到底「有沒有權限建立 Service Account、設定 RBAC」嘛，這就和[昨天](https://ithelp.ithome.com.tw/articles/10350434)介紹的 RBAC 有關嘍。

----

**參考資料**

[service account](https://kubernetes.io/docs/concepts/security/service-accounts/)

[Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

[透過 MicroK8s 認識 Kubernetes 的 Service Account (服務帳戶)](https://blog.miniasp.com/post/2022/08/24/Understanding-Service-Account-in-Kubernetes-through-MicroK8s)

