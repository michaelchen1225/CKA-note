# Day 04 -【Workloads & Scheduling】：Manual Scheduling(上)：nodeName & nodeSelector

### 今日目標

* 使用 nodeName 與 nodeSelector 來調度 Pod 到特定的 Node 上

我們以前談過在 Master Node(controlplane) 中有一個 `kube-scheduler` ，它的工作是負責 cluster 中資源的調配，例如會自動安排 Pod 該放到哪個 Node 上。

在「Storage」的章節中，為了讓 Pod 被調度到有 hostPath 的 Node 上，我們在 yaml 中使用了 `spec.nodeName` 欄位來指定 Node。這種直接手動設定，而不經過 kube-scheduler 的調度，稱為「Manual Scheduling」

一般來說，Manula Scheduling 有以下四大方式： 

1. **nodeName**
2. **nodeSelector**
3. **Affinity**
3. **Taint** 與 **Toleration**

今天，我們先來看看前兩種方式: nodeName 與 nodeSelector。

> 如果你是 single-node cluster，操作同樣能成功，差別在於調度的效果不明顯。你可以到 [killercoda](https://killercoda.com/) 或 [Play with Kubernetes](https://labs.play-with-k8s.com/) 上開個「至少兩個 Node」的環境來跟著操作，這樣效果會比較明顯。

### nodeName

我們先將 kube-scheduler 暫停，然後部署一個 Pod，看看會發生什麼事：

* 暫停 kube-scheduler：
```bash
mv /etc/kubernetes/manifests/kube-scheduler.yaml ~/
```
> 這樣就能暫停 kube-scheduler 了，原理在本章談到「Static Pod」的概念後會再介紹。

* 部署一個 Pod：
```bash
kubectl run test --image nginx
```

* 可以看到 Pod 處於「Pending」的狀態，因為缺少了 kube-scheduler 的調度，而且我們也沒有手動指定 Pod 的去向：
```bash
kubectl get po test
```
```text
NAME   READY   STATUS    RESTARTS   AGE
test   0/1     Pending   0          16s
```

這時，我們就可以用 nodeName 來手動調度。nodeName 顧名思義，就是直接指定 Node 的名字，讓 Pod 在該 Node 上面執行，相當簡單粗暴：

```yaml
# test-nodename.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nodename
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: node01 # 在這裡
```
```bash
kubectl apply -f test-nodename.yaml
```

建立 Pod 後，可以發現即使在沒有 scheduler 的情況下，我們能透過手動調度的方式指定 Node 的去向：

```bash
kubectl get po test test-nodename -o wide
``` 
```bash
NAME            READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
test            0/1     Pending   0          5m49s   <none>        <none>   <none>           <none>
test-nodename   1/1     Running   0          91s     192.168.1.4   node01   <none>           <none>
```

> 你也可以使用 jsonpath 來查看 Pod 所在的 Node:
```bash
kubectl get po test-nodename -o jsonpath="{.spec.nodeName}{'\n'}"
```

使用 nodeName 需要注意以下幾點：

   * 如果指定的 Node 不存在，Pod 就無法正常執行，可能被自動刪除。
   * 如果指定的 Node 沒有足夠的資源來容納 Pod，Pod 也會無法執行。

所以在使用 nodeName 時，要特別注意 Node 的狀態，例如 Node 上的資源是否足夠，還有不要打錯字XD

* 現在把 kube-scheduler 重新跑起來：
```bash
 mv ~/kube-scheduler.yaml /etc/kubernetes/manifests/
```

* 等一段時間後，可以發現先前 Pending 的 test Pod 已成功執行，並且被自動調度到 node01：
```bash
kubectl get po test -o wide
```
```text
NAME   READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
test   1/1     Running   0          11m   192.168.1.5   node01   <none>           <none>
```

### nodeSelector

看到 Selector，不知道有沒有想起什麼？沒錯，就是 `Label`。

我們可以將 Node 標記特殊的 Label，然後在 yaml 中指定 NodeSelector 來指定 Pod 的去向。

相比起 nodeName，nodeSelector 有更多的彈性，例如：

> 今天只是想要將 Pod 放到 Label 為 「has=gpu」的 Node 上，而有這個 Label 的 Node 可能有好幾個，使用 NodeSelector的話，我不用知道這些 Node 到底叫什麼名字，只要知道它們有這個 Label 就好。

不過，我們要如何幫 Node 加上 Label 呢？這時就要使用「kubectl label」指令了：

* kubectl label 指令格式：
```bash
kubectl label <object-type> <object-name> <label-key>=<label-value>
```
* 拿掉某個 Label 的指令則是：

```bash
kubectl label <object-type> <object-name> <label-key>-
```

例如我要為 node01 加上 `has=gpu` 的 Label，可以這樣下指令:
```bash
kubectl label node node01 has=gpu
# 如果要拿掉這個 Label 則是: kubectl label node node01 has-
```

把 Node 加上 Label 後，就可以在 Pod yaml 中使用 nodeSelector：
```yaml
# need-gpu.yaml
apiVersion: v1
kind: Pod
metadata:
  name: need-gpu
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    has: gpu
```

建立 Pod 之後，來 check 一下是否符合需求：
```bash
kubectl apply -f need-gpu.yaml
```
```bash
kubectl get po need-gpu -o wide
```
```text
NAME       READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
need-gpu   1/1     Running   0          15s   192.168.1.4   node01   <none>           <none>
```

### 今日小結

今天介紹了兩種 Manual Scheduling 的方式:  `nodeName` 與 `nodeSelector`。前者直接指定 node 的名字，後者則須搭配 Label 與 nodeSelector 。明天我們來看後面兩種方式：Affinity 與 Taint。

-----
**參考資料**

* [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)

