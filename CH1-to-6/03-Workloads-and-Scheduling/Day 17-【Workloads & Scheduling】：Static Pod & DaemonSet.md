# Day 17-【Workloads & Scheduling】：Static Pod & DaemonSet

### 今日目標

* Static Pod 的概念與實作
  * 如何找到 Static Pod 的目錄
  * 建立一個 Static Pod

* DaemonSet 的概念與實作
  * 如何快速建立 DaemonSet 的 yaml？

在 [Day03](https://ithelp.ithome.com.tw/articles/10345660) 我們使用了 kubeadm 來建立 cluster。現在如果執行：

```bash
kubectl get po -n kube-system
```

你會發現 control-plane 上的重要組件，也就是 kube-apiserver、kube-controller-manager、kube-scheduler、etcd都是以「Pod」的形式執行的，如果嘗試 kubectl delete 也刪不掉，但他們又不是 Deployment 或 ReplicaSet，這是為什麼？

答案是，因為他們是 **Static Pod**。

另外我們也曾經提過， Pod 的建立需要經過 Master Node(control-plane) 的 kube-apiserver 傳遞指令給 kubelet，那問題來了：在 cluster 正在初始化時，kube-apiserver 又還沒開始執行，那為甚麼 control-plane 上的重要元件可以以 Pod 的形式執行呢？難道有某種方式可以不需要 kube-apiserver 的指示，讓 kubelet 自己建立新的 Pod？

答案是「有辦法」，那就是 **Static Pod**。

### Static Pod

由開頭討論的問題中，我們可以發現 Static Pod 的兩個特性：

  * kubectl delete 無法刪除 (刪了會進入 Pending 狀態並立即重建)
  * 不需經過 kube-apiserver ，可由 kubelet 直接建立並執行

底下我們來討論討論這兩個特性。

在之前的介紹中，我們建立 Pod 的手段不外戶就是這兩種：

  * yaml
  * 指令

指令就是使用者藉由 kubectl 與 kube-apiserver 溝通來建立 Pod；換言之，如果 kube-apiserver 掛了， kubectl 就毫無用武之地。

不過，使用 **yaml** 建立的 Static Pod 就能夠避開上述風險。你可能會說：

「透過 yaml 建立 Pod 不是也需要 kubectl create 或 apply，所以還是要經過  kube-apiserver 嗎？」

其實，我們可以在 Node 上預先準備 Pod 的 yaml，並將這些 yaml 集中放置在一個**目錄**，最後將這個目錄路徑告訴 kubelet，kubelet 會自動讀取這個目錄，並將裡面的 yaml 變成成 Pod 執行。

> 既然 yaml 就是放在某個特定的 Node上，所以 Static Pod 也就只會在這個 Node 上執行，並不會經過 schedule 的過程。

Static Pod 被建立後，就直接由該 Node 上的 kubelet 管理。kubelet 會定時查檢查，如果 Pod 終止了，但它的 yaml 放在 Static Pod 目錄中，就會立即重建。

此外，kubelet 會為正在執行的 Static Pod 建立一個 read-only 的鏡像，讓 kube-apiserver 意識到這個 Static Pod 的存在。所以我們能用指令看到 Static Pod 的存在，但無法用指令做編輯(例如 kubectl edit、kubectl set image)。

> 原因是，我們透過指令看到的只是 read-only 的鏡像，而不是 Static Pod 本身。

Static Pod 的鏡像名稱通常用「所在 Node 的名稱」當作後綴(suffix)，例如：
  
```bash
kubectl get po -n kube-system 
```
```text
NAME                                      READY   STATUS    RESTARTS      AGE
etcd-controlplane                         1/1     Running   2 (33m ago)   25d
kube-apiserver-controlplane               1/1     Running   2 (33m ago)   25d
kube-controller-manager-controlplane      1/1     Running   2 (33m ago)   25d
kube-proxy-f8kcp                          1/1     Running   2 (33m ago)   25d
kube-proxy-l9tvl                          1/1     Running   1 (33m ago)   25d
kube-scheduler-controlplane               1/1     Running   2 (33m ago)   25d
```
> 這裡的 Master Node 名為 controlplane，所以帶有「controlplane」後綴的就是 Static Pod (所以 kube-proxy 不是 Static Pod)

### Static Pod 的目錄路徑

剛剛提到，Static Pod 的 yaml 放在一個目錄裡，而這個目錄的指定方式在 kubelet 的設定檔中，我們可以用以下方式來觀察 kubelet 的設定檔：

```bash
ps aux | grep kubelet | grep -- --config
```
```text
root         743  1.4  2.8 1956112 57888 ?       Ssl  12:40   0:04 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml ......
```

這個「/var/lib/kubelet/config.yaml」就是 kubelet 的設定檔，我們來觀察一下其內容：

* 查看 kubelet 的設定檔，找到「staticPodPath」這個參數：

```bash
cat /var/lib/kubelet/config.yaml | grep static
```
輸出：
```text
staticPodPath: /etc/kubernetes/manifests
```

所以，Static Pod 的 yaml 就放在「/etc/kubernetes/manifests」這個目錄中。如果你在 Master Node 上觀察這個目錄的內容：

```bash
ls /etc/kubernetes/manifests
```
```text
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

沒錯！上面的四個 yaml 就是 Master Node 上的四個重要組件。

那如果你需要刪除、修改 Static Pod，就是直接到 /etc/kubernetes/manifests 這個目錄中移除 yaml，或是修改 yaml。同理，如果你需要新增 Static Pod，就是把新的 yaml 放到這個目錄中即可：

* 新增一個 yaml：
```bash
kubectl run static-nginx --image nginx --dry-run=client -o yaml > /etc/kubernetes/manifests/static-nginx.yaml
```

* 觀察新增的 Pod：
```bash
kubectl get po
```
```text
NAME                        READY   STATUS    RESTARTS   AGE
static-nginx-controlplane   1/1     Running   0          12s
```

* 如果嘗試刪除 Static Pod：

```bash
kubectl delete po static-nginx-controlplane
```

會發現 kubelet 又將 Static Pod 重啟了。

所以，當你擔心 kube-apiserver 掛掉會導致無法建立一些特定、重要的 Pod 時， Static Pod 就是一個不錯的解決方案。

> **Tips**

為了讓大家了解 kubelet 的設定檔位置，所以剛才用了較繁瑣的方式來找 Static Pod 的路徑，其實你可以直接用以下指令找到：

```bash
journalctl -u kubelet | grep "static pod"
```

另外，有時候因為不知名的原因，修改 Static Pod 的 yaml 後卻沒有重啟成功，這時可以將 yaml 從原本的路徑搬出去再搬回來，讓 kubelet 重新讀取。

例如：
```bash
mv /etc/kubernetes/manifests/etcd.yaml /tmp
mv /tmp/etcd.yaml /etc/kubernetes/manifests
```

## DaemonSet

Static Pod 可以讓你在特定的 Node 上面穩定的執行某些 Pod，而 DaemonSet 則是讓你在**每一個** Node 上穩定的執行**一個** Pod Instance。

> 在 [Day 16](https://ithelp.ithome.com.tw/articles/10347661) 中，其實就是用 Pod Anti-Affinity 來讓 Deployment 模擬 DaemonSet。 

DaemonSet 會確保每個 Node 上都執行**一個** Pod Instance，如果 Instance 壞掉會嘗試重啟。當某些 Node 新增或刪除時，DaemonSet 會自動調整 Pod 的數量，維持「每個 Node 上都有**一個** Pod Instance」的狀態。

DaemonSet 常見的應用場景如下：
  * 監控程式
  * 日誌收集程式
  * 代理程式 (Agent)

實際的例子就是 kube-proxy。kube-proxy 就是一個 DaemonSet，你可以發現每個 Node 上，不多不少都有一個 kube-proxy 的 Pod：

* 查看有多少個 Node：

```bash
kubectl get node
```
```text
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   25d   v1.30.0
node01         Ready    <none>          25d   v1.30.0
```

* 查看 kube-proxy 的 DaemonSet：

```bash
kubectl -n kube-system get daemonset kube-proxy 
```
```text
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   2         2         2       2            2           kubernetes.io/os=linux   25d
```

* 查看 kube-proxy 的 Pod：
```bash
kubectl get po -n kube-system -o wide | grep kube-proxy
```
```text
kube-proxy-f8kcp                          1/1     Running   2 (52m ago)   25d   172.30.1.2    controlplane   <none>           <none>
kube-proxy-l9tvl                          1/1     Running   1 (52m ago)   25d   172.30.2.2    node01         <none>           <none>
```

### 建立DaemonSet

其實你可以把 DaemonSet 當成 replica 為 1 的 Deployment，所以你可以這樣快速的建立 yaml：

```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx.yaml
```

然後修改以下欄位:
  * **kind** ： Deployment -> DaemonSet
  * **spec.replicas** : 刪掉這行
  * **spec.strategy** : 刪掉這行 (關於這個欄位的說明，可以參考 [Day 7](https://ithelp.ithome.com.tw/articles/10346223))

```yaml
apiVersion: apps/v1
kind: DaemonSet  # Deployment -> DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

最後，就可以用 kubectl apply 來建立 DaemonSet：
  
```bash
kubectl apply -f nginx.yaml
```

確認看看是否每個 Node 都有一個 nginx 的 Pod：

```bash
kubectl get po -l app=nginx -o wide
```
```text
NAME          READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
nginx-d96lj   1/1     Running   0          42s   192.168.0.4   controlplane   <none>           <none>
nginx-rrpfs   1/1     Running   0          42s   192.168.1.4   node01         <none>           <none>
```
### 利用 DaemonSet 在每個 Node 上執行特定作業

我們可以透過 DaemonSet 的特定，在每個 Node 上執行特定的作業，例如在每個 Node 上都寫入一個檔案：

> 在 cluster 中每個 Node 上的 /config/conf 寫入「Hello, Kubernetes!」

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: config
  name: config
spec:
  selector:
    matchLabels:
      app: config
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: config
    spec:
      volumes:
      - name: config
        hostPath:
          path: /config
      containers:
      - image: busybox
        name: busybox
        command: ["/bin/sh", "-c"]
        args: ["echo 'Hello, k8s Day 17！' > /configuration/conf ; sleep 3000"]
        volumeMounts:
        - name: config
          mountPath: /configuration
```
```bash
kubectl apply -f config.yaml
```
> 不清楚 Volume 的概念，可回去參考 [Day 13](https://ithelp.ithome.com.tw/articles/10347182)


* 確認是否在每個 Node 上都有 /config/conf 檔案：

**controlplane**:
```bash
cat /config/conf
```
```text
Hello, Kubernetes!
```

**node01**:
```bash
cat /config/conf
```
```text
Hello, Kubernetes!
```

### 指定 DaemonSet 到特定的 Node

有時候你可能會想要讓 DaemonSet 只存在於符合條件的 Node 上，這時候前面介紹的 nodeSelector、Node affinity、tolerations 就派上用場了。

* nodeSelector : 於 yaml 中的「`spec.template.spec.nodeSelector`」欄位指定
* affinity : 於 yaml 中的「`spec.template.spec.affinity`」欄位指定
* tolerations : 於 yaml 中的「`spec.template.spec.tolerations`」欄位指定

> 詳細說明及原理可以參考 [Day 15](https://ithelp.ithome.com.tw/articles/10347495)、[Day 16](https://ithelp.ithome.com.tw/articles/10347661)，這裡就不再贅述。


### 今日小結

今天介紹了 Static Pod 與 DaemonSet，兩者都能讓你穩定的在 Node 上執行 Pod，但應用場景不同：

  * Static Pod : 用於在特定 Node 上執行 Pod，且不需擔心 api-server 壞掉的情況。
  * DaemonSet : 用於在每一個 Node 上都執行一個 Pod

-----
**參考資料**

* [Create static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)