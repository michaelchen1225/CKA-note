## 【Basic Concept】：Namespace

## 目錄

* [什麼是 Namespace？](#什麼是-namespace)

* [預設的四大 Namespace](#預設的四大-namespace)

* [建立 Namespace](#建立-namespace)

* [namespace 的基本操作](#namespace-的基本操作)

### 什麼是 Namespace？

假如一個公司讓所有部門共用同一個 cluster，可能會有一些問題：

   * 部門 A 不小心刪除了部門 B 的資源
   * 部門 A 的服務不小心與部門 B 的服務撞名
   * 資源的分配不均，導致部門 A 的服務佔用了部門 B 的資源

你可能會說：那就讓每個部門都有自己的 cluster 不就好了？但是，這樣可能會造成資源的浪費，而且溝通與資料共享並不方便。其實除了建立個別的 cluster 之外，使用「Namespace」就足以解決上述面臨的問題。

我們可以這樣描述 Namespace：

> 一個**邏輯**上的「空間」，將不同的資源分門別類。

Namespace 的主要功能有：

* **隔離資源**

例如之前提過的 Pod、Deployment 等等。例如 A 部門的 Pod 放進 A 部門的 namespace，B 部門的 Pod 放進 B 部門的 namespace。

* **分配資源的使用量**：
   
透過 Resource Quota 或 Limit Range 來管理或限制每個 Namespace 的資源使用量，例如 CPU、Memory 等等。(關於 Resource Quota & Limit Range 的介紹，我們會在「WorkLoads & Scheduling」的章節再做介紹)

* **限制資源的存取權限**
   
可以限制不同 namespace 的存取權限，例如讓 A 部門的員工無法存取 B 部門的 Pod。(這我們會在「Cluster Architecture, Installation & Configuration」的章節中，談到 RBAC 時再做介紹)

所以 Namespace 的作用並不只有「分類」這麼簡單，還可以透過各種設定達到「分隔」的效果。如果你只是想做出**分類**的效果，之前介紹的 label 就非常夠用了。例如區分軟體版本，就不用特的劃分 namespace，只要在 label 中加入 version=xxx 即可。

要注意的是，並不是所有的資源都能用 namespace 劃分，例如 Node 、Persistent Volume 等等。

> 你可以用指令來查看哪些資源可以放在 namespace 中：

```bash
kubectl api-resources --namespaced=true
```

### 預設的四大 Namespace

事實上，k8s 預設已經存在了四個 namespace，分別是：

* **default**

預設的 namespace，如果操作中沒有指定 namespace，就是使用 default。例如我們之前的各種練習，都是在這個 namespace 中進行的。

* **kube-system**

k8s 自己創建的物件都會放在這裡，通常是一些維持 cluster 運作的重要物件。如果你是用 kubeadm 建置 cluster，你可以在這個 namespace 底下找到以 Pod 執行的 kube-api-server、kube-controller-manager 等重要物件。

* **kube-public**

這個 Namespace 可以被所有 client 端（包括未經身份驗證的 client）存取到。主要用來存放一些 cluster 的公用資源，不過這只是一個慣例，並沒有強制要求。

* **kube-node-lease**

存放每個 Node 的 lease 物件。lease 主要是用來協助檢查 Node 是否還活著。


### 建立 Namespace

你同樣可以使用 yaml 或是 kubectl create 來建立 namespace，例如要建立一個名為 **dev** 的 namespace，可以這樣做：

* 建立 dev-namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

* 建立 namespace：
```bash
kubectl apply -f dev-namespace.yaml
```

* 如果是用 kubectl create 創建 namespace就更簡單了：
```bash
kubectl create namespace dev
```

### namespace 的基本操作

* 列出所有 Namespace：
```bash
kubectl get namespace
```

* 列出所有 namespace 中的 Pod：
```bash
kubectl get pod --all-namespaces
```
或是:
```bash
kubectl get pod -A
```

* 列出特定 namespace 的 Pod：

```bash
kubectl -n <ns-name> get pod 
```

> **範例**：查看 kube-system namespace中的 Pod
```bash
kubectl -n kube-system get pod
```

*  在特定 namespace 中建立一個 Pod：

```bash
kubectl -n <ns-name> run <pod-name> --image <image-name>
```

* 在 yaml 中指定 namespace，將物件放入特定的 namespace 中：
```yaml
apiVersion: <api-version>
kind: <object-kind>
metadata:
  namespace: <namespace-name>
  name: <object-name>
spec:
...(省略)...
```

例如在 dev namespace 中建立一個 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: dev
  name: my-dev-pod
spec:
   containers:
   - name: my-dev-pod
      image: nginx
```

其實你應該發現了，很多時候要在在特定 namespace 中執行 kubectl，就是加上參數 -n 就對了 :

```bash
kubectl -n <ns-name> <kubectl-cmd>
```

如果你很常在特定的 namespace 活動，不想要每次下指令時都要指定，不妨更改預設的 Namespace：

```bash
kubectl config set-context --current --namespace=<ns-name>
```

> 這裡的設定意義與 kubeconfig 有關，我們會在「Services & Networking」的章節中再做介紹。

### 補充：刪除 Namespace

若要刪除一個 Namespace ，得先確認是否「該 Namespace 底下的資源已經都沒用、不需要了」，因此建議先將該 Namespace 的資源全部列出來確認：

```bash
kubectl get all -n <ns-name>
```

如果確認了 Namespace 底下的資源都不需要了，就可以刪除所有資源：

```bash
kubectl delete all --all -n <ns-name>
```

然後再刪除 Namespace：

```bash
kubectl delete namespace <ns-name> --force --grace-period=0
```

不過此時你會發現被刪除的 Namespace 卡在 Terminating 的狀態，原因是存在「Finalizer」的機制：

> **Finalizer**：通常存在具有「parent-child」關係的物件上，防止 parent 物件被刪除後留下 orphaned child 的情況。

以 Namespace 為例，Namespace 是 parent 物件，而 Namespace 底下的資源（Pod、Deployment 等等）就是 child 物件。當一個 Namespace 被建立時，就會自動被加上 Finalizer，所以在`kubectl delete namespace` 後，還需要把 Finalizer 移除，才能真正刪除 Namespace：

* Namespace 卡在 Terminating 的狀態：

```bash
kubectl get ns calico-system 
```
```
NAME            STATUS        AGE
calico-system   Terminating   386d
```
```bash
kubectl get ns calico-system -o yaml | grep -A 1 -m 1 finalizer
```
```
  finalizers:
  - kubernetes
```

* 移除 Finalizer：

```bash
# 先啟動 kubectl proxy，讓我們可以直接對 API Server 發出請求
kubectl proxy --port=8080 &
```

```bash
cat <<EOF | curl -X PUT \
  localhost:8080/api/v1/namespaces/calico-system/finalize \
  -H "Content-Type: application/json" \
  --data-binary @-
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "calico-system"
  },
  "spec": {
    "finalizers": null
  }
}
EOF
```

* 確認 Namespace 已經被刪除：

```bash
kubectl get ns calico-system 
```
```
Error from server (NotFound): namespaces "calico-systme" not found
```

> 補充一個 shell script 來刪除所有 Terminating 的 Namespace (Ubuntu 環境)：

```bash
#!/bin/bash

# Check if jq is installed
if ! command -v jq &> /dev/null; then
    echo "jq not found. Installing jq..."
    sudo apt-get update
    sudo apt-get install -y jq
else
    echo "jq is already installed."
fi

echo "Checking for namespaces stuck in Terminating state..."

for ns in $(kubectl get ns | grep Terminating | awk '{print $1}'); do
    echo "Force deleting namespace: $ns"

    kubectl get namespace "$ns" -o json \
        | jq '.spec.finalizers=[]' \
        | kubectl replace --raw "/api/v1/namespaces/$ns/finalize" -f - > /dev/null

done

echo "Finished cleaning Terminating namespaces."
```


### 今日小結

其實 Namespace 的操作並不難，所以今天的內容會相對輕鬆些，因為指定 Namespace 就是加上 -n 參數，不然就是在 yaml 中的 metadata.namespace 欄位指定。

雖然操做簡單，不過 Namespace 卻是相當常用的概念，熟悉自己 cluster 內部的 Namespace 並做好規劃，能讓管理的工作變得事半功倍。

-----
**參考資料**
* [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

* [Configure Memory and CPU Quotas for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

