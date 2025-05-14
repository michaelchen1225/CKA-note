## 【Basic Concept】：ReplicaSet、Deployment & StatefulSet

## 目錄

* [Pod scaling](#pod-scaling)

  * [ReplicSet](#replicset)

  * [Label](#label)

  * [Deployment](#deployment)

  * [Deployment 中的 ReplicaSet](#deployment-中的-replicaset)

  * [StatefulSet](#statefulset)

---

在前面兩天了解了 Pod 的概念與操作後，我們來想像一個情況: 假設服務僅由一個 Pod 來提供，可能會發生那些情況？

1. 如果今天這個 Pod 因為某個原因掛掉了，那使用者就無法取得服務，即使 Pod 順利重啟，但還是造成了等待時間。

2. 如果今天網路流量變高，但因為只有一個 Pod，它無法負荷這麼多的使用者，進而導致服務效率變低，甚至可能造成服務崩潰。

而這些情況都是我們想極力避免的。這時，**Pod scaling** 的概念就派上用場了。那什麼是 Pod scaling？

## Pod scaling

簡單來說，就是讓 K8s 增減 Pod 的數量，以達到我們的需求。

例如當流量變多時，我們可以讓 K8s 增加 Pod 的數量，以負擔更多的使用者。而當流量變少時，我們也可以讓 K8s 減少 Pod 的數量，以節省資源。且多個 Pod 能夠提高容錯率，因為當某個 Pod 掛掉時，至少其他 Pod 仍可以提供服務。

而我們可以透過今天的主題，也就是 ReplicSet、Deployment 與 StatefulSet，來實現 Pod scaling。

> 上述三者皆能實現 Pod scaling，不過除此之外他們都有各自的特性與應用場景，接下來會逐一介紹。

### ReplicSet

從英文字面上直接解讀，ReplicaSet 就是**複製品的集合**：

  * 複製品：以一個 **Pod** 為樣本 (template) 複製多個。

那我們究竟需要「多少個」複製品，就是所謂的 **desired number**。

當我們建立 ReplicaSet 時，會告訴 K8s 複製品的 template 以及指定的 desired number，k8s 就會負責確保 cluster 中的複製品數量與我們指定的 desired number 相同。

但在一個 cluster 中有那麼多 Pod， K8s 是如何辨哪些是 ReplicaSet 的複製品，好進行維護？

另外，如果我們有多個 ReplicaSet，各自都有不同的 desired number，那 k8s 該如何區分呢？這裡就必須來了解一下「Label」的概念了。

### Label

Label 是一個「key-value」的組合，主要用來：

   * **識別物件**：例如我們可以為一個 Pod 加上「app: nginx」的標籤。透過 K8s 的「Label Selector」功能，就可以輕鬆的在整個 cluster 中找到這個Pod。

   * **分類物件**：假如系統中存在著有關「前端」與「後端」的 Pod，我們可以為前端應用加上「tier: frontend」的 Label，而後端應用加上「tier: backend」，同樣可以用 Label Selector 來分別找到前端或後端的 Pod。

**Label 在 yaml 中的設定**

在 K8s 中幾乎所有的資源都能被賦予 Label，只需在 yaml 中的「metadata.labels」中設定即可。

* 建立一個帶有 Label 的 Pod：

```yaml
# label-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-test
  labels:      # 在這裡設定 Label
    app: nginx 
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
``` 
> 共設定了兩個 Label: app=nginx, tier=frontend


* 建立 Pod：

```bash 
kubectl apply -f label-test.yaml
```

* 查看 Pod 的 Label：

```bash
kubectl get pod label-test --show-labels
```
```text
NAME         READY   STATUS    RESTARTS   AGE   LABELS
label-test   1/1     Running   0          70s   app=nginx
```

* 使用 Label Selector 篩選出標籤為「app: nginx」的 Pod： 
```bash
kubectl get pod -l app=nginx
```
```text
NAME         READY   STATUS    RESTARTS   AGE
label-test   1/1     Running   0          4m11s
```

---
> **Tips：使用 kubectl 對 Label 進行操作**

* 同樣可以用 kubectl run 建立一個帶有 Label 的 Pod：
```bash
kubctl run label-test --image nginx -l app=nginx,tier=frontend
```

* 用 kubectl label`來為已經存在的物件加上 Label:
```bash
kubectl label <object-type> <object-name> <key>=<value>
```

* 使用多個 Label 篩選：
```bash
kubectl get pod -l <key1>=<value1>,<key2>=<value2>,...
```
***

ReplicaSet 就是透過 Label 來**識別**需要複製與維護的 Pod 。我們來看一個 ReplicaSet 的範例 yaml：

```yaml
# nginx-rs.yaml
apiVersion: apps/v1 # 是 apps/v1，而不是 v1
kind: ReplicaSet 
metadata:
  name: nginx-rs
spec:
  replicas: 3      # 自行指定的 desired number
  selector:        # Label Selector
    matchLabels:
      app: nginx
  template:        # Pod 的樣本，以此為樣本產生複製品
    metadata:
      labels:
        app: nginx # 樣本的 Label 要與上面spec.selector.matchLabels相同
    spec:
      containers:
      - name: nginx
        image: nginx
```

* 建立 ReplicaSet：

```bash
kubectl apply -f nginx-rs.yaml
```

* 查看 ReplicaSet，是否符合我們設定的 desired number？

```bash
kubectl get rs
```
```text
NAME       DESIRED   CURRENT   READY   AGE
nginx-rs   3         3         3       10s
```

這樣就建立一個名為 **nginx-rs** 的 ReplicaSet，目前的 replica 有 3 個 (CURRENT)，目前可用的也有 3 個 (READY)，並且 k8s 會將複製品數維持在我們希望的 **3** (DESIRED)。

我們可以透過 Label selector 來看到這些 replica :
```bash
kubectl get pod -l app=nginx
```
```text
NAME             READY   STATUS    RESTARTS   AGE
nginx-rs-86kng   1/1     Running   0          90s
nginx-rs-jd9kk   1/1     Running   0          90s
nginx-rs-pnjnl   1/1     Running   0          90s
```

如果嘗試刪除其中一個 Pod，k8s 會自動重建一個新的 Pod，以維持 desired number：

```bash
kubectl delete pod nginx-rs-86kng
```

```bash
kubectl get pod -l app=nginx
```
```text
NAME             READY   STATUS    RESTARTS   AGE
nginx-rs-b5sld   1/1     Running   0          12s  # New Pod
nginx-rs-jd9kk   1/1     Running   0          2m6s
nginx-rs-pnjnl   1/1     Running   0          2m6s
```

不過，單純的維持數量並不是 Pod scaling 的精隨，畢竟顧名思義，它是「 Pod 的**增減**」。既然是「增減」複製品的數量，當然就可以調整 desired number：

* 調整 desired number：kubectl scale
```bash
kubectl scale rs <rs_name> --replicas <new_desired_number>
```

* 例如將剛剛建立的 nginx-rs 的 desired number 增加至 5
```bash
kubectl scale rs nginx-rs --replicas 5
```

* 當然也可以漸少至 2
```bash
kubectl scale rs nginx-rs --replicas 2
```

### Deployment

Deployment 可以看成是功能更強大的 ReplicaSet，ReplicaSet 有的功能它都有，而且多了一個新功能：**Rolling Update**，也就是滾動式更新。

當樣本 Pod 的 image 需要更新時， Deployment 會**逐步**汰換舊的複製品。這樣有順序的汰換過程，可以確保除了正在更新的 Pod 之外，其他的 Pod 仍可以提供服務，達成 `Zero Downtime`(零停機更新)。

> 這裡僅簡述 Rolling Update 的概念，其他操作將在明天提到。

馬上來看一個 Deployment 範例：

* 建立一個 Deployment，名為 **nginx-deploy**，desired number 為 3，template 使用 nginx image：

```yaml
# nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment # 其實和 ReplicaSet 的 yaml 幾乎一樣，就改 kind 和名字而已 
metadata:
  name: nginx-deploy
spec:
  replicas: 3  # desired number
  selector:  
    matchLabels:
      app: nginx
  template:  
    metadata:
      labels:
        app: nginx  
    spec:
      containers:
      - name: nginx
        image: nginx
```

* 建立 Deployment：
```bash
kubectl apply -f nginx-deploy.yaml
```

> 你也可以用指令建立 Deployment：
```bash
kubectl create deployment nginx-deploy --image nginx --replicas 3
```

(想建立 Deployment 的 yaml？--dry-run -o yaml 是你的好幫手)


* 看看 Deployment 狀況如何：
```bash
kubectl get deploy nginx-deploy
```
```text
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           113s
```

* 同樣也可以用 Label Selector 來看到這些 replica :
```bash
kubectl get pod -l app=nginx
```
```text
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-7854ff8877-mx7jg   1/1     Running   0          20s
nginx-deploy-7854ff8877-np7wm   1/1     Running   0          20s
nginx-deploy-7854ff8877-tdllg   1/1     Running   0          20s
```

* 同樣可以調整 desired number：
```bash
kubectl scale deploy nginx-deploy --replicas 5
```

* 試試看更新 image：
```bash
kubectl set image deployment nginx-deploy nginx=nginx:1.14.2
#注意: 是 <容器名稱>=<新image>，並不是<舊image>=<新image>                   
```

* describe 看看是不是真的更新了：
```bash
kubectl describe deployment nginx-deploy
``` 
```bash
......
Pod Template:
  Labels:  app=nginx-deploy
  Containers:
   nginx:
    Image:        nginx:1.14.2 # 更新成功!
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
...(省略)...
```

### Deployment 中的 ReplicaSet

事實上，當你建立一個 Deployment，會自動建立 ReplicaSet 來負責維持 desired number。它們的關係如下圖：

![https://ithelp.ithome.com.tw/upload/images/20230926/20161118emE0mmLEST.jpg](https://ithelp.ithome.com.tw/upload/images/20230926/20161118emE0mmLEST.jpg)

利用指令也可以看出這個關係：
```bash
kubectl get deploy
```
```text
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   5/5     5            5           13m
```
```bash
kubectl get rs
``` 
```text
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-5bbf5d957f   5         5         5       8m42s
nginx-deploy-846d6f46b7   0         0         0       13m 
```
> 因為剛剛更新過一次 image，所以 AGE 較老的 replicaSet 是上一個版本遺留的。

### StatefulSet

在介紹 StatefulSet 之前，我們先來了解一下「Stateful vs Stateless」的概念。

* **Stateful**：顧名思義就是「有狀態的」，這些「狀態」是需要被保存的，代表一次的操作可能會影響到應用的後續狀態。例如資料庫、IoT 應用對於裝置的設定紀錄等等。

* **Stateless**：則是「無狀態的」，這些應用不會保存任何狀態，每次的操作都是獨立的。例如 Web Server，每次的 request 都是丟回來一個同樣的網頁，上次的 request 對下次的 request 沒有影響。

容器有一個特性，就是生命週期短，所以不適合用來保存狀態，所以恰恰適合 Stateless 的應用，例如 Web Server、API 等等。

但是如果要用容器來部署 Stateful 的應用，例如資料庫，我們通常會使用外部的儲存空間掛載到容器中，以防容器掛掉時資料丟失。

那麼在 K8s 中，我們也能對 Pod 加讓外部儲存空間來保存資料，也可以對這些 Pod 指定 desired number，而這樣的管理方式就是 StatefulSet。

總之，如果你希望你的應用服務有以下特性，就可以考慮使用 StatefulSet：

1. **Stable, unique network identifiers**：應用服務擁有穩定且獨立的網路識別，不會像普通的 Pod 一樣在重啟後會 IP 變動。

2. **Stable, persistent storage**：每個 Pod 都有自己的儲存空間，不會因為重啟而資料丟失。

3. **Ordered, graceful deployment and scaling.**：應用服務的部署是有序的，一個 template 建好了才換下一個，不像 Deployment 同時部署多個 template。

4. **Ordered, automated rolling updates.**：更新也是有序的，一個 template 更新完才換下一個(這點和 Deployment 一樣)。


簡單來說，如果是 Stateful 的應用，就用 StatefulSet 來部署。如果是 Stateless 的應用，就用 Deployment 來部署即可。

要注意的是，用來保存狀態的外部儲存空間，例如 PersistentVolume、PersistentVolumeClaim、StorageClass 等等，都要在建立 StatefulSet 前設定好。

> 關於 PersistentVolume、PersistentVolumeClaim、StorageClass 的介紹，將在「Storage」章節中提到。這裡先看懂就好，主要是了解 StatefulSet 的概念。

我們來看一個官網的 StatefulSet [範例 yaml](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#creating-a-statefulset)：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx" # Stable, unique network identifiers
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates: # Stable, persistent storage
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

其實大致上與 Deployment 的 yaml 類似，同樣用 Label Selector 來識別 template，並且可以指定 desired number。

與 Deployment 不同的是，為了讓 template 有獨立且穩定的網路識別，StatefulSet 還要指定 serviceName，並且將 volumeClaimTemplates 中設定的外部儲存空間「www」掛載到每個 template 的 /usr/share/nginx/html，以保存該目錄的資料。

> yaml 中 serviceName 和 Service 的概念有關，這裡先簡單理解成外界可透過 nginx 這個 Service 來穩定的存取 template。關於 Service 的介紹會在後面的「【Basic Concept】：Service」中提到。

當這個 StatefulSet 建立後，K8s 會「依序」建立 3 個 template，並且為每個 template 都產生一組固定的識別資訊，在重啟後確保 template 以前的狀態會被保留。

另外，當我們用 Label Selector 查看這些 template 時，會發現他們的後綴名稱是有規則的，例如 web-0、web-1、web-2，而不向 Deployment 那樣是隨機的後綴名稱：

```bash
NAME   READY   STATUS    RESTARTS   AGE
web-0  1/1     Running   0          20s
web-1  1/1     Running   0          20s
web-2  1/1     Running   0          20s
```

> 由於我們還沒有介紹 Service 與 Storage 相關的概念，這裡就不實際建立 StatefulSet 了，只要先了解 StatefulSet 的概念即可。之後等介紹完 Service 與 Storage 後，會在附錄補充實際操作。

### 今日小節

今天從 Pod scaling 的概念展開討論，帶出了 ReplicaSet 與 Deployment 的基本用途與簡單的範例，最後介紹了 StatefulSet 的概念，說明 stateful 與 stateless 的應用場景。

Deployment 算是一個相當常用的重要資源，因為實務上不可能單用一個 Pod 來提供服務，所以盡量在本章熟悉 Deployment 的操作，例如：

* 如何用指令建立 Deployment
* 如何更新 image
* 如何調整 desired number
* 如何快速建立 Deployment 的 yaml 樣本等等。

-----
**參考資料**

* [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
* [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#components)
* [StatefulSet Basics](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#creating-a-statefulset)
* [Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
* [Stateful vs. Stateless Applications: What’s the Difference?](https://blog.purestorage.com/purely-educational/stateful-vs-stateless-applications-whats-the-difference/)



