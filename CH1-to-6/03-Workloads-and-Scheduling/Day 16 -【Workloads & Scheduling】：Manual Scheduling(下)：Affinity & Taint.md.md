# Day 16 -【Workloads & Scheduling】：Manual Scheduling(下)：Affinity & Taint.md

### 今日目標

* 了解並使用 Affinity/Anti-affinity 來調度 Pod
  * Node affinity/anti-affinity
  * Inter-pod affinity/anti-affinity

* 了解並使用 Taint & Tolerations Pod

如果用 label 的方式來指定 Pod 的去向，前一章介紹的 nodeSelector 是**最簡單**的方式。但如果你想在 Node 的篩選條件中加入更多候選人、或是較彈性的篩選強制性，nodeSelector 就無法達成了。

今天我們來看另一種「更彈性」的方式，也就是 Affinity/Anti-affinity。


### Affinity/Anti-affinity

「Affinity」翻譯成中文是「親和力」，以 Pod Scheduling 的角度來說，越有親和力的 Node 越可能被選來執行 Pod。那哪些 Node 會被視為有親和力呢？可以透過「Node 或 Pod 的 Label」進行篩選：

  * **Node affinity** : 透過「Node 的 Label」來篩選出特定的 Node，讓 Pod 被安排到符合條件的 Node 上。

  * **Inter-pod affinity** : 透過「Pod 的 Label」來篩選出特定的 Pod，然後在指定的 Node topology 中找看看有沒有這些特定的 Pod，如果有才會把新的 Pod 安排到該 Node topology 中執行。

  * **Inter-pod anti-affinity** : 透過「Pod 的 Label」來篩選出特定的 Pod，然後在指定的 Node topology 中找看看有沒有這些特定的 Pod，如果有，則會將新的 Pod 放到**其他**的 Node topology 中執行。

>  什麼是 Node topology？可以參考「[附錄]()」


而「Label 的篩選條件」則由以下三者組成：

   * **key**：Node/Pod label 的 key
   * **value**：Node/Pod label 的 value。一個 key 可能對應到多個 value ，所以在這個欄位中可以填入一個或多個 value
   * **operator**：可以是 In、NotIn、Exists、DoesNotExist、Gt、Lt，代表含意如下表：

| Operator | 作用 | 
| -------- | ----------- |
| In | Node/Pod label 的 value 等於**values**欄位中的其中一個 |
| NotIn | Node/Pod label 的 value 不等於**values**欄位中的任何一個 |
| Exists | **key**欄位指定的 Node/Pod key 存在 |
| DoesNotExist | **key**欄位指定的 Node/Pod key 不存在 |
| Gt | Node label 的 value 大於**values**欄位中的其中一個 |
| Lt | Node label 的 value 小於**values**欄位中的其中一個 |

**注意**

> 1. 如果 operator 是 Exists 或 DoesNotExist，**不用**設定 values 欄位，因為這兩個 operator 是以 key 是否存在為判斷依據，與 value 無關！

> 2. 只有 Node Affinity 可以使用 Gt、Lt，Inter-pod Affinity/Anti-affinity 不支援這兩個 operator。另外，在使用 Gt、Lt 時，Label value 會被轉成「整數數值」來比較。如果 value 不能被轉成數值，Pod 會啟動失敗。

> 3. 之所以沒有 Node Anti-affinity，是因為 Node Anti-affinity 的**效果**其實透過設定 In、NotIn 即可達成。


假如篩選結果中沒有任何 Node 雀屏中選，難道 Pod 就會一直 Pending 嗎？這要看你選擇哪種強制性來設定 Affinity/Anti-affinity：

**RequiredDuringSchedulingIgnoredDuringExecution**

在 Pod 被安排到 Node 時， 要求 Node **必須**符合篩選條件(Require During Scheduling)，否則 Pod 就會 Pending。但這個條件並不影響已在執行的 Pod (Ingnored During Execution)

**PreferredDuringSchedulingIgnoredDuringExecution**

在 Pod 被安排到 Node 時，scheduler 會**盡可能**找出符合篩選條件的 Node，但如果找不到合適的 Node，仍會安排 Pod 到其他 Node 上。(Preferred During Scheduling)，但這個條件並不影響已在執行的 Pod (Ingnored During Execution)。

另外 preferred 需要設定「weight(權重)」，範圍是 1~100，當有多個 Node 符合篩選條件時，scheduler 會將 weight 與 node priority function 一起計算，分數最高的 Node 會被選來執行 Pod。

> 總而言之，你可以先選擇篩選結果的強制性(Required/Preferred)，再看要用 Node 還是 Pod 的 Label 來篩選(Node Affinity or Pod Affinity/Anit-Affinity)。這樣的設定方式比起 nodeSelector 來說更有彈性。

### Node Affinity 的 yaml 寫法

Node affinity 會寫在 Pod yaml 中的 **spec.affinity.nodeAffinity** 欄位中：

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: <node-label-key>
            operator: In   # In、NotIn、Exists、DoesNotExist、Gt、Lt
            values:   # 可以填入一個或多個 value
            - <node-label-value1>
            - <node-label-value2>
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: <1-100>
        preference:
          matchExpressions:
          - key: <node-label-key>
            operator: Exists
```

> 上面的設定中，required 和 preferred 可以同時設定，也可以只設定其中一個。另外，preferred 底下示範的是 operator 是 Exists 的情況，所以不用填入 values 欄位。(如果你想用 In、NotIn，寫法就和上面 required 的一樣，需設定 values)

> 需要小心的是，required 和 preferred 底下的設定乍看之下寫法類似，但卻有些許的不同。筆者在一開始初學時，到官網上 copy 了 required 的設定，但後來發現要的其實是 preferred 的效果，還天真的以為只要把「requiredDuring....」改成「preferredDuring...」並加上 weight 就行了，結果一直報語法錯誤，看了半天才發現原來 required 與 preferred 的設定方式是不同的...... 

我們來看一個例子：
```yaml
...(省略)
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tier
            operator: In
            values:
            - frontend
            - backend
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: developer
            operator: Exists
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

上面的例子中，設定了兩個 Node Affinity 條件：

1. **requiredDuringSchedulingIgnoredDuringExecution**:

   目標 Node 的 label **key** 必須有「tier」，且 **value** 必須是「frontend」或「backend」。


2. **preferredDuringSchedulingIgnoredDuringExecution**:

   希望目標 Node 的 label **key** 有「developer」，而 **value** 隨意。但如果找不到符合的 Node ，仍會安排 Pod 到其他 Node 上。

所以，目標 Node **一定要有**「tier=frontend」或「tier=backend」，如果有「developer=xxx」的 Node 會被優先考慮，不過這不是必要條件。

### Node Affinity 實作

> 如果是 single-node cluster，可以到 killercoda 上開一個環境來跟著練習：

* 首先，幫 node01 加上 label：

```bash
kubectl label node node01 app=nginx
```

* 創建一個 Pod yaml：
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx.yaml
```

* 編輯 yaml，加入 affinity 的設定，讓 Pod 被安排至 node01：

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
```bash
kubectl apply -f nginx.yaml
```

* 檢查 Pod 是否再 node01 上：

```bash
kubectl get po nginx -o wide
```
```text
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          37s   192.168.1.4   node01   <none>           <none>
```

### Pod Affinity/Anti-affinity 的 yaml 寫法

Pod Affinity/Anti-affinity 會寫在 Pod yaml 的 **spec.affinity.podAffinity** 或 **spec.affinity.podAntiAffinity** 欄位中：

```yaml
...(省略)
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: <pod-label-key>
            operator: In # In、NotIn、Exists、DoesNotExist
            values:
            - <pod-label-value1>
            - <pod-label-value2>
        topologyKey: <topology-key>
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: <1-100>
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: <pod-label-key>
              operator: In
              values:
              - <pod-label-value-1>
              - <pod-label-value-2>
          topologyKey: <topology-key>
```

> 同樣的，podAffinity 與 podAntiAffinity 可以同時設定，也可以只設定其中一個。required 與 preferred 也可以隨意搭配 podAffinity 與 podAntiAffinity。

關於 **topologyKey** 的作用就是畫分 Node Topology。

>  什麼是 Node topology？可以參考「[附錄]()」

### Pod Anti-affinity 實作

我們到 [killercoda](https://killercoda.com/playgrounds)上開一個「兩個 Node」的環境，嘗試部署一個 Deployment，但讓每個 Pod template 分散在不同 Node 上執行：

* 首先，查看兩個 node 上的 Label：

```bash
kubectl get node --show-labels
```
```text
AME           STATUS   ROLES           AGE   VERSION   LABELS
controlplane   Ready    control-plane   31d   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controlplane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
node01         Ready    <none>          31d   v1.30.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01,kubernetes.io/os=linux
```

> 可以發現每個 Node 上都有一個 label 叫做「kubernetes.io/hostname」的 key，而且 value 都不同，這正好可以用來當作 topologyKey，達到讓 Pod 分散在不同 Node 上執行的效果。

* 創建一個 Deployment yaml：

```bash
kubectl create deploy nginx --image=nginx --replicas=3 --dry-run=client -o yaml > nginx-deploy.yaml
```

* 編輯 yaml，加入 podAntiAffinity 的設定，讓 Pod 分散在不同 Node 上：

```yaml
# nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy: {}
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
      affinity:                                             
        podAntiAffinity:                                    
          requiredDuringSchedulingIgnoredDuringExecution:   
          - labelSelector:                                  
              matchExpressions:                             
              - key: app                                     
                operator: In                                
                values:                                     
                - nginx                          
            topologyKey: kubernetes.io/hostname 
status: {}
```
```bash
kubectl apply -f nginx-deploy.yaml
```

* 觀察一下 Pod 的分佈：

```bash
kubectl get po -o wide -l app=nginx
```
```text
NAME                     READY   STATUS    RESTARTS   AGE    IP            NODE           NOMINATED NODE   READINESS GATES
nginx-5f5fc7b8b9-6s59g   1/1     Running   0          112s   192.168.0.5   controlplane   <none>           <none>
nginx-5f5fc7b8b9-hdnxs   0/1     Pending   0          112s   <none>        <none>         <none>           <none>
nginx-5f5fc7b8b9-xq2zk   1/1     Running   0          112s   192.168.1.6   node01         <none>           <none>
```

> 可以發現 Pod 已分散在 controlplane 與 node01 上執行。且因為 nginx-deploy.yaml 中 replicas 設定為 3，所以當其中兩個 template 跑起來後，第三個因為 podAntiAffinity 的設定，只能 Pending 等待新的 Node Topology 出現，因為目前所有的 Node Topology 都已經有 Label 為「app=nginx」的 Pod 在跑了！

(如果要避免上面一個 template 跑不起來的狀況，可以改用 preferredDuringSchedulingIgnoredDuringExecution。這裡用 required 是為了讓效果比較明顯)

所以，使用 Pod Anti-Affinity 搭配 topologyKey，可以讓 Pod 分散在不同的 Node 上執行，達到分散資源利用的效果。

### Taint & Tolerations

Node Affinity 是讓 Pod 經由篩選 Label 的方式指定 Node，而 Taint & Tolerations 則是讓 Node 可以「避免不符條件的 Pod 被安排到自己身上」。怎麼說呢? 我們用一個「疫苗」的比喻來說明：

> 疫苗的作用是讓人體產生抗體，來抵抗病毒的入侵。但不能保證我們百毒不侵。例如小明施打了疫苗，站在一般病毒的視角而言，小明被疫苗「汙染(Taint)」了，所以它不會來找麻煩。不過特別的 X 病毒有可能找上小明，因為該疫苗不能抵抗 X 病毒，原因是X病毒能夠「容忍(Toleration)」被該疫苗。

所以 Taint & Tolerations 的概念就是：

我們為一個 Node (小明) 設定「taint(疫苗)」後，只有能夠「tolerate(容忍)」這些 taint 的 Pod (病毒)才能被安排到這個 Node 上。

所以在設定上，taint 會被設定在 Node 上，而 toleration 則設定在 Pod 上。一般來說為了確保 master node 的獨立性，預設會有一個 taint：

```bash
kubectl describe node master | grep -i taint
```
```text
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

如此一來，只要 Pod 沒有相對應的 `toleration`，就不會被安排到 master node 上，進而確保了 master node 的穩定與獨立性。

---

**注意**

> 如果你的練習環境是 single-node cluster 或 killercoda，master node 上的 taint 已經被移除了，所以上述查看 taint 的指令會找不到任何結果。

***

### Taint的設定方式

Taint 的設定對象是 Node，並且可以選擇三種效果(effect)：

1. **NoSchedule** : 

除非 Pod 有相對應的 toleration，否則不會被安排到這個 Node 上。但**不影響**已在該 Node 上執行的 Pod。

2. **PreferNoSchedule** : 

除非 Pod 有相對應的 toleration，否則不會被安排到這個 Node 上，不過前面多了一個**偏好**(prefer)，代表如果真的找不到符合的 Node ，還是會安排 Pod 到 Node 上。

3. **NoExecute** : 

除非 Pod 有相對應的 toleration，否則不會被安排到這個 Node 上，且該效果還會影響**正在執行**的 Pod，如果 Pod 沒有相對應的 toleration，會被驅逐(evict)出 Node ; 如果有相對應的 toleration，則會繼續**留在** Node 上執行。

> 至於會留下多久，則是由 toleration 的 `tolerationSeconds` 欄位決定。這個後面會說明。

了解了 taint 的效果後，我們來看看 taint 的設定方式：

* 設定 taint：
```bash
kubectl taint node <node-name> <key>=<value>:<effect>
```

* 在某個 key 上設定 taint，但不指定 value：
```bash
kubectl taint node <node-name> <key>:<effect>
```

* 移除 taint：
```bash
kubectl taint node <node-name> <key>=<value>:<effect>-
```

* 移除某個 key 下特定 effect 的 taint：
```bash
kubectl taint node <node-name> <key>:<effect>-
```

* 移除同一個 key 下的所有 taint：
```bash
kubectl taint node <node-name> <key>-
```

例如我要將 node01 設定一個 `app=blue` 的 taint，效果是 NoSchedule：
```bash
kubectl taint node node01 app=blue:NoSchedule
```

如果想知道一個 Node 上的所有 taint，可以用 describe 或 jsonpath 來查詢：
```bash
kubectl describe node <node-name> | grep -i taint -A 10
```
或是：
```bash
kubectl get node <node-name> -o jsonpath='{.spec.taints}'
```

### Toleration 的設定方式

Toleration 的設定對象是 Pod，可以在 yaml 中的**spec.tolerations**底加入以下欄位來設定：

* **key**：目標 taint 的 key

* **value**：目標 taint 的 value

* **operator**：可以是 Exists、Equal，無指定預設為 Equal，如果寫的是Exists，則不需填寫 value 欄位

* **effect**：目標 taint 的效果，也就是 NoSchedule、PreferNoSchedule、NoExecute

* **tolerationSeconds**：如果 effect 是 NoExecute，我們才會考慮設定這個欄位：

  * 有設定 tolerationSeconds：若某 Pod 還在執行，當它可容忍的 taint 被加入到其所在 Node 上時，會留在該 Node 幾秒後才會被刪除。

  * 沒有設定 tolerationSeconds：若某 Pod 還在執行，當它可容忍的 taint 被加入到其所在 Node 上時，則可以繼續留在該 Node 上執行，不受影響。

假如我要幫一個名為 nginx 的 Pod 加入一個 `app=blue` 的 toleration，效果是NoSchedule：
```yaml
......
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

再來看一個有設定 `tolerationSeconds` 的例子：
```yaml
......
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoExecute"
    tolerationSeconds: 300
```

> 上面的設定表示，如果 nginx Pod 正在某 Node 上執行，我們對該 Node 加入了 `app=blue` 的 taint，效果是 NoExecute，那麼 nginx Pod 會在 300 秒後，除非把該 Node 上的 taint 移除，否則 nginx Pod 會被驅逐出 Node。

所以說，如果不想要 NoExecute 的 taint 影響到正在執行的 Pod，就單純的讓 Pod 可以容忍該 taint 即可，不要設定 tolerationSeconds。

### 特別的 Toleration 設定方式

1. 讓 Pod 可以容忍**所有的** taint:
```yaml
......
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: ""
    operator: Exists
```

2. 讓 Pod 可以容忍某 key 下所有 Effect 的 taint:
```yaml
......
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "app"
    operator: "Exists"
    effect: ""
```
### 問題思考

以上為 taint & toleration 的基本觀念與設定方式，這裡我們來思考三個問題：

> **Q1**：假如某 Node 上設置了一堆的 taint，而我讓某個 Pod 只容忍其中一個 taint，這個 Pod 會被安排到這個 Node 上嗎？

不會，Pod 必須容忍該 Node 上**所有的** taint，才「有可能」被安排到這個 Node 上。

> **Q2**：如果一個 Pod 能夠容忍一個 Node 的 taint，這會保證 Pod 一定會被安排到這個 node 上嗎？

當然不會，這只是保證 Node 不會接納不合條件的 Pod ，並不代表符合條件的 Pod 一定會被安排 Node 上。你可以這樣想想：小明打了流感疫苗就代表他一定會得流感嗎？

> **Q3**：我們昨天介紹了 nodeName 與 nodeSelector，再加上今天的 node affinity、taint 與 toleration，如果今天設置了互相衝突的規則，那究竟該聽誰的？

除非 taint 的效果是 NoExecute，否則 nodeName 最大，再來是 taint，最後才是 nodeSelector 與 node affinity。舉例來說，我們用「nodeName」指定某 Pod 到帶有 taint 的  node01，但該 Pod 並沒有設定 toleration，但是 Pod 仍可以在 node01 上執行。

### Node Affinity、Taint & Tolerations 的搭配

我們可以用 toleration 與 affinity 的搭配，**確保** Pod 只能被安排指定的 Node 上，**而且**該 Node 不會接受來路不明的 Pod。

我們的需求如是：

> 「backend」Pod 只能被安排到「tier=backend」的 Node 上，且 node01 只會接受 backend Pod。

* 首先，幫 node01 設定一個 tier=backend 的 `taint`，效果是 NoExecute：
```bash
kubectl taint node node01 tier=backend:NoExecute
```

* 然後為 node01 加上 label：
```bash
kubectl label node node01 tier=backend
```

* 創建一個 backend Pod 的 yaml：
```bash
kubectl run backend --image=redis --dry-run=client -o yaml > backend.yaml
```

* 修改 yaml 檔，加入 affinity 與 toleration 的設定：
```yaml
# backend.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: backend
  name: backend
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tier
            operator: In
            values:
            - backend    
  tolerations:
  - key: "tier"
    value: "backend"
    effect: "NoExecute"
  containers:
  - image: redis
    name: backend
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
```bash
kubectl apply -f backend.yaml
```

* 查看 Pod 是否在 node01 上：
```bash
kubectl get po backend -o wide
```
```text
NAME      READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
backend   1/1     Running   0          5m30s   192.168.1.5   node01   <none>           <none>
```

### 今日小結

今天介紹另外兩種 scheduling 的方式：Affinity 與 Taint。 Affinity 相較於 Node selector 更為彈性且應用更多元。而 Taint 則是讓 Node 可以避免不符條件的 Pod 被安排到自己身上。由於觀念上比較抽象，所以加入了「小明打疫苗」的比喻來說明 taint & toleration 的概念。

-----
**參考資料**

[Assign Pods to Nodes using Node Affinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)

[Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)