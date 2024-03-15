# *Day9 Scheduling* : `nodeName`與`nodeSelector`

我們在[Day02 Basic Concept*: `Kubernetes`的架構與組件](02.md)，談過在`control plane`中有一個`kube-scheduler`的組件，它的工作是負責`cluster`中資源的調配，例如哪些`Pod`該放到哪個`Node`上。

而在之前的練習中，我們並沒有指定`Pod`要放到哪個`Node`上，所以`kube-scheduler`會根據特定的演算法，自型決定`Pod`的去向。

不過，有時我們會想要**自行**安排`Pod`到特定的`Node`上，這時可以使用這幾種方式:
* 在`yaml`中使用`nodeName`
* 使用`nodeSelector`
* 使用`Taint`與`Toleration`
* 使用`Node Affinity`

今天，我們先來看看前兩種方式: `nodeName`與`nodeSelector`。

# nodeName

顧名思義，就是藉由在`yaml`中指定`Node`的名字，來決定`Pod`的去向，十分簡單粗暴。

我們直接來看一個`yaml`範例:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: node01 # 就在這裡直接打上Node的名字即可
```

建立`pod`之後，可以在`kubectl get po`加上` -o wide`的參數來查看`pod`的資訊，底下可以看到`pod`已經被安排到`node01`上了:
```bash
$ kubectl apply -f nginx.yaml

$ kubectl get po -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          10s   10.244.1.2   node01   <none>           <none>
```
使用`nodeName`的就是這麼簡單! 不過在使用時，需要注意以下幾點:
   * 如果指定的`Node`不存在，`pod`就無法正常運行，甚至可能被自動刪除
   * 如果指定的`Node`沒有足夠的資源來容納`pod`，`pod`也會無法運行

所以在使用`nodeName`時，要特別注意`Node`的狀態，例如`Node`上的資源是否足夠，還有不要打錯字!

# nodeSelector

看到`Selector`，不知道你有沒有想起什麼? 沒錯，就是`Label`! 我們可以將`Node`賦予特殊的`Label`，然後在`yaml`中指定`NodeSelector`，再來`kube-scheduler`就會根據`NodeSelector`來決定`Pod`的去向。

> 例如，在一家企業的`cluster`運行的A服務需要GPU，那麼我們可以在有GPU的`Node`上加上`Label`，例如`has: gpu，然後在A服務的`yaml`中指定`NodeSelector`為`has: gpu`，這樣`kube-scheduler`就會將`Pod`安排到有GPU的`Node`上。

不過，我們要如何幫`Node`加上`Label`呢? 這時就要使用`kubectl label`指令了:

`kubectl label指令格式`:
```bash
$ kubectl label <object-type> <object-name> <label-key>=<label-value>
# 不只可以用來label Node，也可以用來label Pod、Deployment等等
```
取消`Label`的指令則是:
```bash
$ kubectl label <object-type> <object-name> <label-key>-
```

例如我要為node01加上`gpu: yes`的`Label`，可以這樣下指令:
```bash
$ kubectl label node node01 has=gpu
# 如果要拿掉這個`Label`則是: kubectl label node node01 has-
```
馬上來看一個範例yaml:
```yaml
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
建立`pod`之後，來check一下是否符合需求:
```bash
$ kubectl apply -f need-gpu.yaml

$ kubectl get po need-gpu -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
need-gpu   1/1     Running   0          95s   10.244.1.6   node01   <none>           <none>

$ kubectl get nodes --show-labels | grep has=gpu
node01         Ready    <none>          73m   v1.27.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,has=gpu,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01,kubernetes.io/os=linux
```

### 練習1: 使用`NodeName`與`NodeSelector`

首先，將一個`Node`加上`Label`，需求如下:
* **node01**: `tier: frontend`

接著，建立兩個`Pod`，分別使用`NodeName`與`NodeSelector`指定去向。需求如下:
   1. 第一個`Pod`:
      * `Name`: `nginx`
      * `Image`: `nginx`
      * `NodeSelector`: `tier: frontend`

   2. 第二個`Pod`:
      * `Name`: `httpd`
      * `Image`: `httpd`
      * `NodeName`: `node01`

**參考**

* [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

