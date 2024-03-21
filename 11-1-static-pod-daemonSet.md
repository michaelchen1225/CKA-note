# *Day11 Scheduling* : `Static Pod`與`DaemonSet`

在[Day03](03.md)我們使用了`kubeadm`來建立`cluster`。現在如果執行:

```bash
kubectl get po -n kube-system
```

你會發現`control-plane`上的重要組件，也就是`kube-apiserver`、`kube-controller-manager`、`kube-scheduler`、`etcd`都是以`pod`的形式運行的，如果嘗試`kubectl delete`也刪不掉，但他們又不是`Deployment`，這是為什麼? 

答案是，因為他們是`Static Pod`。

另外我們也曾經提過，`Pod`的建立需要經過`control-plane`的`kube-apiserver`傳遞指令給`kubelet`，那問題來了: 在`cluster`正在建立時，`kube-apiserver`又還沒開始運行，那為甚麼`control-plane`上的重要元件可以以`pod`的形式運行呢? 難道有某種方式可以不需要`kube-apiserver`的指令，就可以讓`pod`運行嗎?

答案是「有辦法」，那就是`Static Pod`。

## What's Static Pod

由開頭討論的問題中，我們可以發現`Static Pod`的其中兩個特性:
  * `kubectl delete`無法刪除
  * 不需經過`kube-apiserver`即可運行

底下我們來討論討論這兩個特性:

[Day04](04-1.md)提到，建立`Pod`的手段可以分為:
  * yaml
  * 指令

而指令就是`kubectl`，透過與`kube-apiserver`溝通來建立`Pod`; 換言之，如果`kube-apiserver`掛了，`kubectl`就毫無用武之地了。

所以，使用**yaml**建立的`Static Pod`就能夠避開上述風險。你可能會說: 

> 「透過yaml不是也需要kubectl，所以還是要經過`kube-apiserver`嗎?」

答案是: 不一定。

在`node`上，我們可以預先準備`pod`的yaml，並將這些yaml集中放置在一個目錄，最後將這個目錄路徑告訴`kubelet`，而`kubelet`會自動讀取這個目錄，並將裡面的yaml變成成`pod`運行。

> 既然yaml就是放在某個特定的node上，所以`Static Pod`也就只會在這個node上運行，並不會經過schedule的過程。

`Static Pod`被建立後，就直接由該`node`上的`kubelet`管理。`kubelet`會定時查檢查，如果`Pod`終止了但有yaml放在指定的目錄中，就會立即重建。

此外，`kubelet`會建立一個read-only的鏡像，告知`kube-apiserver`這個`Static Pod`的存在。我們能用指令看到`Static Pod`的存在，但卻無法刪除、用指令做編輯(例如`kubectl edit`、`kubectl set image`)。

> 原因是，我們看到的只是read-only的鏡像，而不是`Static Pod`本身。

`Static Pod`的鏡像名稱通常以所在`node`的名稱結尾，例如:
  
```bash
$ kubectl get po -A
controlplane $ k get po -n kube-system 
NAME                                      READY   STATUS    RESTARTS      AGE
etcd-controlplane                         1/1     Running   2 (33m ago)   25d
kube-apiserver-controlplane               1/1     Running   2 (33m ago)   25d
kube-controller-manager-controlplane      1/1     Running   2 (33m ago)   25d
kube-proxy-f8kcp                          1/1     Running   2 (33m ago)   25d
kube-proxy-l9tvl                          1/1     Running   1 (33m ago)   25d
kube-scheduler-controlplane               1/1     Running   2 (33m ago)   25d
# controlplane結尾的就是Static Pod
```
## Static Pod的路徑

剛剛提到，`Static Pod`的yaml放在一個目錄裡，而這個目錄的指定方式在`kubelet`的設定檔中，我們可以用以下方式來觀察`kubelet`的設定檔:

* 首先，確認`kubelet`的啟動腳本位置:
```bash
$ systemctl status kubelet.service 
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
```

可以看到`kubelet`的啟動腳本在"/lib/systemd/system/kubelet.service.d/10-kubeadm.conf"

* 接著，查看`kubelet`的設定檔在哪裡:

```bash
$ cat /lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml" # 這行就是指定kubelet的設定檔位置
```
> 這裡再介紹一個快速的方式找到`kubelet`的設定檔:

```bash
ps aux | grep kubelet | grep -- --config
root         743  1.4  2.8 1956112 57888 ?       Ssl  12:40   0:04 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml ...(省略)
```

* 最後，查看`kubelet`的設定檔，找到"staticPodPath"這個參數:

```bash
$ cat /var/lib/kubelet/config.yaml
..
...(省略) 
staticPodPath: /etc/kubernetes/manifests
..
...(省略)
```

所以，`Static Pod`的yaml就放在`/etc/kubernetes/manifests`這個目錄中。
如果你在`control-plane`上觀察這個目錄的內容:

```bash
$ ls /etc/kubernetes/manifests
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```
沒錯! 上面的四個yaml就是`control-plane`上的`Static Pod`。

那如果你需要刪除，修改`Static Pod`，就是直接到`/etc/kubernetes/manifests`這個目錄中刪掉yaml，或是修改yaml。同理，如果你需要新增`Static Pod`，就是把新的yaml放到這個目錄中即可。

> 所以，當你擔心`kube-apiserver`掛掉會導致無法建立特定的`Pod`時，`Static Pod`就是一個不錯的解決方案。

**Tips**

有時候因為不知名的原因，修改`static pod`的`yaml`後，卻沒有重啟成功，這時可以將`yaml`從原本的路徑搬出去再搬回來，讓`kubelet`重新讀取。

例如:
```bash
mv /etc/kubernetes/manifests/etcd.yaml /tmp
mv /tmp/etcd.yaml /etc/kubernetes/manifests
```

## DaemonSet

如果今天你要部署一個監控程式到**每一個**`node`上，你可以慢慢的手動在每個`node`，或是使用`DaemonSet`。

### What's DaemonSet

`DaemonSet`會確保每個`node`上都不多不少的運行**一個**`Pod Instance`，而且當有`node`新增或刪除時，`DaemonSet`會自動調整`Pod`的數量，維持「每個`node`上都有**一個**`Pod Instance`」的狀態。

DaemonSet常見的應用場景如下:
  * 監控程式
  * 日誌收集程式
  * 存儲代理程式

實際的例子就是`kube-proxy`(在[Day02](02-1.md)介紹過)，`kube-proxy`就是一個`DaemonSet`，你可以發現每個`node`上，不多不少都有一個`kube-proxy`的`Pod`:
  
```bash
$ kubectl get node
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   25d   v1.29.0
node01         Ready    <none>          25d   v1.29.0

$ kubectl -n kube-system get daemonset kube-proxy 
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   2         2         2       2            2           kubernetes.io/os=linux   25d

$ kubectl get po -n kube-system -o wide | grep kube-proxy
kube-proxy-f8kcp                          1/1     Running   2 (52m ago)   25d   172.30.1.2    controlplane   <none>           <none>
kube-proxy-l9tvl                          1/1     Running   1 (52m ago)   25d   172.30.2.2    node01         <none>           <none>
```

### 建立DaemonSet

其實你可以把DaemonSet當成replica為1的Deployment，所以你可以快速的建立DaemonSet的yaml:

```bash
$ kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx.yaml
```

然後修改以下欄位:
  * `kind` : `Deployment` -> `DaemonSet`
  * `spec.replicas` : 刪掉這行
  * `spec.strategy` : 刪掉這行

```yaml
apiVersion: apps/v1
kind: DaemonSet
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

最後，就可以用`kubectl apply`來建立`DaemonSet`:
  
```bash
$ kubectl apply -f nginx.yaml
```

### 指定DaemonSet到特定的Node

有時候你可能會想要指定`DaemonSet`到符合特定的`node`上，這時候前面學過的`nodeSelector`、`affinity`、`tolerations`就派上用場了。

* nodeSelector : 於yaml中的`spec.template.spec.nodeSelector`欄位指定
* affinity : 於yaml中的`spec.template.spec.affinity`欄位指定
* tolerations : 於yaml中的`spec.template.spec.tolerations`欄位指定

> 詳細說明及原理可以參考前面章節，這裡就不再贅述。

### 練習1: 建立一個Static Pod
建立一個'Static Pod'，需求如下:
  * `name`: static-httpd
  * `image`: httpd

### 練習2: 更新`Static Pod`
將練習1建立的`Static Pod`的`image`更新為`httpd:alpine`

### 練習3: 建立一個`DaemonSet`
建立一個`DaemonSet`，需求如下:
  * `name`: nginx
  * `image`: nginx