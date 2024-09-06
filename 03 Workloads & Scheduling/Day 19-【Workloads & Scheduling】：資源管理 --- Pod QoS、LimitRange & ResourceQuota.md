### 今日目標

* 資源管理的三大方式：
  * Pod QoS (Quality of Service)
  * LimitRange：限制 namespace 底下單一 Pod 的資源用量
  * ResourceQuota：限制 namespace 底下的總資源用量

我們知道一個 Node 上的資源有限，當一個 Pod 占用過多的資源，或是因為資源不足而導致 Pod 無法正常運作，都是我們不樂見的。今天就來介紹一些管理資源的方式，來有效的控管、分配資源。

所謂「資源」，指的是 CPU 和 memory：

  * CPU；基本單位可以是：
    * **cores**：1 core = 1000m 
    * **millicores**: 1000m = 1 core
  
  * Memory：基本單位是：
    * **Ki** (Kibibyte)：2^10 bytes
    * **Mi** (Mebibyte)：20 bytes
    * **Gi** (Gibibyte)：2^30 bytes

在 k8s 中，我們可以在 yaml 中用 `requests` 和 `limits` 來指定 Pod 使用的資源範圍：

  * **requests**：保證 Pod 能夠獲得的最小資源量。當「Node 可提供的資源 >= requests」時，Pod 才會被分配到該 Node 上。

  * **limits**：用來限制 Pod 能夠使用的最大資源量。

例如，下面我們定義了一個 nginx Pod，最少需要 200Mi 的 memory 和 700m 的 CPU，最多能使用 400Mi 的 memory 和 800m 的 CPU：
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources: 
      limits:
        memory: "400Mi"
        cpu: "800m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

那麼在管理資源(CPU、Memory)上，我們有以下三種方式：

1. Pod QoS
2. LimitRange
3. ResourceQuota

### Pod QoS 的三種等級

Pod QoS 將不同的 requests 和 limits 設定，劃分以下三種不同的 QoS class(類別)

   * **Guaranteed**
   * **Burstable**
   * **BestEffort**

對管理者來說，當 Node 上的資源不夠時，可以用 Pod 的 Qos class 來「**預估**」哪些 Pod 大概率會先刪除。

> 理論上，當資源不夠時會先從 BestEffort 開始刪，再來是 Burstable，最後是 Guaranteed。

> 若 Qos class 為同一等級，則按照以下兩個原則來刪除：

   1. 資源使用量超過 requests 的 Pod 先刪
   2. 占用最多資源的 Pod 先刪

例如現在我想知道整個 cluster 中所有 Pod 的 QoS class，來判斷資源吃緊時，大概會先刪除哪些 Pod，可以用以下指令：

```bash
kubectl get po -A -o jsonpath="{range .items[*]}{.metadata.name}   {.status.qosClass}{'\n'}"
 ```

**Tips**：

* 上面提到的「刪除」是比較通俗的講法，正式的名稱是 `evict`(驅逐)，指的是當 Node 上的資源不足時，讓一個或多個 Pod 被 Terminate 的過程。

* 要注意的是，Qos Class 只能用來「預估」大概的 eviction 順序，因為該順續並非由 Qos Class 決定，這在[官網](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/#kubelet-%E9%A9%B1%E9%80%90%E6%97%B6-pod-%E7%9A%84%E9%80%89%E6%8B%A9)中有明確的提到:

> The kubelet **does not** use the pod's QoS class to determine the eviction order. You can use the QoS class to estimate the most likely pod eviction order when reclaiming resources like memory.

* 真正決定驅逐順序的是以下三者：

1. Pod 占用的資源是否高過 requests。
2. [Pod Priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)：值越高，優先級越高，越不容易被驅逐，同時也會被優先安排到 Node 上。
3. Pod 相對於 requests 的資源使用量。

> 詳細的說明可參考[官網說明](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

* Pod 的 priority，可以這樣查看：

```bash
kubectl get po <pod-name> -o yaml | grep -i priority
```

了解了 Pod QoS 的概念後，接下來我們來看看實際上要如何設定不同的 class。

### Guaranteed

將一個 Pod 設定為 Guaranteed 等級，須滿足以下所有條件：

1. 每個 Pod 中的容器(container)都需要有 CPU request 和 limit，且 **request=limit**
2. 每個  Pod  中的容器都需要有 memory request和limit，且 **request=limit**

以下為一個 Guaranteed Pod 的範例 yaml：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: qos-guaranteed
    image: nginx
    resources:
      limits:
        cpu: "500m"
        memory: "300Mi"
      requests:
        cpu: "500m"
        memory: "300Mi"

```

**提醒**：如果你只有設定 cpu 或 memory 的 「limit」，但沒有設定相對應的 request，k8s 會在創建 Pod 時自動給予和 limit 值相同的 request 值。

### Burstable

將一個 Pod 設定為 Burstable 等級，須滿足以下所有條件：

1.	在 Pod 中**至少有一個**容器有 memory 或 cpu 的 request 或 limit。
2.	這個 Pod 並**不滿足** Qos Guaranteed 的條件。

以下為一個 Burstable Pod 的範例 yaml：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: qos-burstable
    image: nginx
    resources:
      limits:
        memory: "400Mi"
      requests:
        memory: "200Mi"
```

### BestEffort

將一個 Pod 設定為 BestEffort 等級，須滿足以下所有條件：

1. 這個 Pod 中**沒有設定任何 memory/cpu request或limit**。

以下為一個 BestEffort Pod 的範例 yaml：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: qos-besteffort
    image: nginx
```


### 查看 Pod 的 QoS class

* 查看單一 Pod 的 QoS class:

```bash
kubectl describe pod <pod-name> -n <namespace> | | grep -i "qos class"
```
或是:
```bash
kubectl get pods <pod-name> -n <namespace> -o jsonpath='{.status.qosClass}'
```

* 查看所有 Pod 的 QoS class:

```bash
kubectl get pods -A -o jsonpath="{range .items[*]}{.metadata.name}   {.status.qosClass}{'\n'}"
```

### LimitRange

不過每個 Pod 都要一個一個設定 request 或 limit ，多少有些麻煩。如果你想讓某個 namespace 的 Pod 建立時都有「預設基本的資源限制」，可以使用 LimitRange。

舉例來說，我想讓 `default` namespace 的 Pod 在建立預設都有 800m 的 cpu request 和 800m 的 cpu limit，可以這樣設定:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-cpu
spec:
  limits:
  - default:
      cpu: 800m
    defaultRequest:
      cpu: 800m
    type: Container
```

建立 LimitRange 後，**對已存在的 Pod 並不會有影響**。若新的 Pod 沒有特別指定 cpu request 或 limit，兩者就會自動設定為 800m。不過在這樣的設定下，若使用者有自己指定 request 或 limit，則不會被限制，例如使用者可以自行指定 cpu request 為 900m。

如果想真對使用者「可設定的資源範圍」進行限制，可以這樣設定：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-cpu-with-range
spec:
  limits:
  - default:
      cpu: 800m
    defaultRequest:
      cpu: 800m
    max:
      cpu: 800m
    min:
      cpu: 200m
    type: Container
```

這樣一來，在 default namespace 中，若使用者有自己指定 request 或 limit，則 request 不得低於 200m，limit 不得高於 800m。若使用者沒有特別指定 cpu request 或 limit，兩者就會自動設定為 800m。

### 測試 LimitRange

我們建立上面名為「default-cpu-with-range」的 LimitRange 後，來進行測試：

* 建立 LimitRange 後，用 describe 可以清楚看到設定的內容：

```bash
kubectl describe limitrange default-cpu-with-range
```
```text
Type        Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---   ---------------  -------------  -----------------------
Container   cpu       200m  800m  800m             800m           -
```

* 建立一個 Pod，request 和 limit 都在可設定的範圍內：
  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-1
spec:
  containers:
  - name: nginx-1
    image: nginx
    resources:
      limits:
        cpu: "700m"
      requests:
        cpu: "400m"
```

```bash
kubectl apply -f nginx-1.yaml 
```

* Pod 會被成功建立，建立後 cpu request 和 limit 為：

```bash
kubectl get pod nginx-1 -o custom-columns=NAME:.metadata.name,REQUEST:.spec.containers[*].resources.requests.cpu,LIMIT:.spec.containers[*].resources.limits.cpu
```
```text
NAME      REQUEST   LIMIT
nginx-1   400m      700m
```

* 清除 nginx-1：
```bash
kubectl delete pod nginx-1
```

* 建立一個 Pod，request 在範圍內，但沒有設定 limit：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-2
spec:
  containers:
  - name: nginx-2
    image: nginx
    resources:
      requests:
        cpu: "400m"
```
```bash
kubectl apply -f nginx-2.yaml
```

* Pod 會建立成功，建立後 cpu request 和 limit 為：

```bash
kubectl get pod nginx-2 -o custom-columns=NAME:.metadata.name,REQUEST:.spec.containers[*].resources.requests.cpu,LIMIT:.spec.containers[*].resources.limits.cpu
```
```text
NAME      REQUEST   LIMIT
nginx-2   400m      800m
```

* 清除 nginx-2：

```bash
kubectl delete pod nginx-2
```

* 建立一個 Pod，不設定 cpu request 或 limit 的 Pod：

```bash
kubectl run nginx-3 --image=nginx
```

Pod 會被成功建立，建立後 cpu request 和 limit 為：

```bash
kubectl get pod nginx-3 -o custom-columns=NAME:.metadata.name,REQUEST:.spec.containers[*].resources.requests.cpu,LIMIT:.spec.containers[*].resources.limits.cpu
```
```text
NAME      REQUEST   LIMIT
nginx-3   800m      800m
```

* 清除 nginx-3：

```bash
kubectl delete pod nginx-3
```

* 建立一個 Pod，但 request 低於範圍：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-4
spec:
  containers:
  - name: nginx-4
    image: nginx
    resources:
      limits:
        cpu: "700m"
      requests:
        cpu: "100m"
```
```bash
kubectl apply -f nginx-4.yaml
```

* Pod 會建立失敗，錯誤訊息如下:

```text
Error from server (Forbidden): error when creating "1.yaml": pods "nginx-4" is forbidden: minimum cpu usage per Container is 200m, but request is 100m
```

## ResourceQuota

LimitRange 是針對一個 namespace 底下的**單一** Pod 進行資源限制，而 ResourceQuota 則是針對一個 namespace 底下所有資源用量的**總和**進行限制。我們直接來看一個範例：

* 建立一個新的 namespace：

```bash
kubectl create namespace test-ns
```

* 建立一個 resource quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
  namespace: test-ns
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```
```bash
kubectl apply -f my-quota.yaml -n test-ns
```

---
> my-quota 的效果:

1. 每個在 test-ns namespace 中的 Pod 必須都要有 cpu 與 memory 的 request 和 limit
2. 在 test-ns 中，所有在 Pod 的「cpu request」總和不能超過 1 cpu
3. 在 test-ns 中，所有 Pod 的「memory request」總和不能超過 1Gi
4. 在 test-ns 中，所有 Pod 的「cpu limit」總和不能超過 2 cpu
5. 在 test-ns 中，所有 Pod 的「memory limit」總和不能超過 2Gi
***

* 接著創建一個 Pod 來測試：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-pod
  namespace: test-ns
spec:
  containers:
  - name: quota-pod
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
```

```bash
kubectl apply -f quota-pod.yaml
```

* 這個 Pod 會被創建成功，因為它的 request 和 limit 都在 quota 的範圍內:

```bash
kubectl describe resourcequota my-quota -n test-ns
```
```text
Name:            my-quota
Namespace:       test-ns
Resource         Used   Hard
--------         ----   ----
limits.cpu       800m   2
limits.memory    800Mi  2Gi
requests.cpu     400m   1
requests.memory  600Mi  1Gi
```

* 我們再創建一個 Pod，來測試如果超過 quota 會發生什麼:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-pod-2
  namespace: test-ns
spec:
  containers:
  - name: quota-pod-2
    image: nginx
    resources:
      limits:
        memory: "1Gi"
        cpu: "800m"
      requests:
        memory: "700Mi"
        cpu: "400m"
```
```bash
kubectl apply -f quota-pod-2.yaml
```

* 當你嘗試建立這個 Pod 時，你會發現這個 Pod 會建立失敗，因為已經超過了 resource quota 的 memmory 限制 (600Mi + 700Mi > 1Gi)：

```text
Error from server (Forbidden): error when creating "3.yaml": pods "quota-pod-2" is forbidden: exceeded quota: my-quota, requested: requests.memory=700Mi, used: requests.memory=600Mi, limited: requests.memory=1Gi
```

* 修改後再次嘗試：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-pod-2
  namespace: test-ns
spec:
  containers:
  - name: quota-pod-2
    image: nginx
    resources:
      limits:
        memory: "400Mi"
        cpu: "800m"
      requests:
        memory: "400Mi"
        cpu: "400m"
```
```bash
kubectl apply -f quota-pod-2.yaml
# 成功建立
```

### 今日小結

今天介紹了三種管理資源的方式: Pod QoS、LimitRange 和 ResourceQuota。我們可以用 Pod QoS 預估 Pod 的 eviction 順序，用 LimitRange 來限制新 Pod 的資源使用範圍，用 ResourceQuota 來限制 namespace 底下所有 Pod 的資源用量總和。

**參考資料**

[Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)

[LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)

[Configure Memory and CPU Quotas for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)