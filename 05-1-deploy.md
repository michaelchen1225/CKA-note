# *Day05 Basic Concept* : `ReplicSet` 與 `Deployment`

我們來想像一個情況: 當整個`cluster`中只有一個Pod，可能會發生那些情況?

例如:
* 如果今天這個Pod因為某個原因掛掉了，那使用者就無法取得服務。
* 如果今天有很多的使用者使用服務，但因為只有一個Pod，它無法負荷這麼多的使用者，進而導致服務效率變低。

而這些情況都是我們想極力避免的。這時，`Pod scaling`的概念就派上用場了。那什麼是`Pod scaling`?

## Pod scaling
簡單來說，就是讓`K8s`可以自動的增減Pod的數量，以達到我們的需求。
例如當流量變多時，我們可以讓`K8s`自動增加Pod的數量，以負單更多的使用者。而當流量變少時，我們也可以讓`K8s`自動減少Pod的數量，以節省資源。

了解完`Pod scaling`的概念後，今天的主題，也就是`ReplicSet`與`Deployment`，就是`K8s`實現`Pod scaling`的兩個方式。

## ReplicSet
從字面上直接解讀，`ReplicaSet`就是**複製品的集合**。那麼是什麼的**複製品**呢? 就是Pod啦。那麼為什麼說是**集合**? 因為不只有一個複製品嘛。

那究竟需要多少的複製品，就是所謂的`desired number`，而`K8s`會確保cluster中的複製品數量與我們指定的`desired number`相同。

但在一個cluster中，有那麼多的Pod，`K8s`是如何辨哪些是`ReplicaSet`的複製品，好進行維護? 另外，如果我們有多個`ReplicaSet`，各自都有不同的`desired number`，那`k8s`該如何區分呢? 這裡就必須來了解一下`Label`的概念了。

## Label
`Label`是一個`key-value`的組合，主要用來 : 
   * **識別物件**: 例如我們可以為一個Pod加上`app: nginx`的標籤。透過`K8s`的`Label Selector`功能，就可以輕鬆的在整個`cluster`中找到這個Pod。
   * **分類物件**: 例如系統中存在這關於前端應用與後端應用的`Deployment`，我們可以為前端應用加上`tier: frontend`的標籤，而後端應用加上`tier: backend`的標籤。這樣我們就可以透過`Label Selector`來分別找到前端應用與後端應用。

**Label範例 : label-test.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-test
  labels: # Pod的標籤
    app: nginx # key: value
spec:
  containers:
  - name: nginx
    image: nginx
``` 
* 建立Pod
```bash 
$ kubectl apply -f label-test.yaml
```
**補充**

> 當然了，你也可以用`kubectl run`建立一個帶有Label的Pod:
```bash
$ kubctl run label-test --image nginx --labels app=nginx
```
> 你也可以用`kubectl label`來為已經存在的Pod加上Label:
```bash
$ kubectl label pod label-test app=nginx
# kubectl label 指令格式: kubectl label <object-type> <object-name> <key>=<value>
```

* 查看Pod的Label
```bash
$ kubectl get pod label-test --show-labels
NAME         READY   STATUS    RESTARTS   AGE   LABELS
label-test   1/1     Running   0          70s   app=nginx
```

* 使用`Label Selector`篩選出標籤為`app: nginx`的Pod
```bash
$ kubectl get pod -l app=nginx
NAME         READY   STATUS    RESTARTS   AGE
label-test   1/1     Running   0          4m11s
```

所以說，`ReplicaSet` 就是透過`Label`來**識別**需要複製的Pod。我們來看一個`ReplicaSet`的範例yaml:
```yaml
apiVersion: apps/v1 # 注意: 這裡是 apps/v1，而不是 v1
kind: ReplicaSet 
metadata:
  name: nginx-rs
spec:
  replicas: 3  # 我們指定的複製品數量，也就是desired number!
  selector:  # Label Selector
    matchLabels:
      app: nginx
  template:  # Pod 的樣本，以此樣本產生複製品
    metadata:
      labels:
        app: nginx  # 注意: 樣本的 Label 要與上面 Selector 的 matchLabel 相同，這樣才能辨識
    spec:
      containers:
      - name: nginx
        image: nginx
```
* 建立`ReplicaSet`
```bash
$ kubectl apply -f nginx-rs.yaml
```

* 查看`ReplicaSet`，是否符合我們設定的`desired number`?
```bash
$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
nginx-rs   3         3         3       10s
```

這樣就建立一個名為**nginx-rs**的`ReplicaSet`，並且`k8s`會將複製品數維持在我們指定的**3**個。

不過，單純的維持數量並不是`Pod scaling`的精隨，畢竟顧名思義，它是「Pod的**增減**」。既然是「增減」複製品的數量，當然就可以調整`desired number`。

我們可以用以下指令調整`desired number`:
```bash
kubectl scale rs <rs_name> --replicas <new_desired_number>
```

* 例如將剛剛建立的`nginx-rs`的`desired number`增加至5
```bash
kubectl scale rs nginx-rs --replicas 5
```

* 當然也可以漸少至2
```bash
kubectl scale rs nginx-rs --replicas 3
```

### Deployment
`Deployment`可以看成是功能更強大的`ReplicaSet`，`ReplicaSet`有的功能它都有，而且多了一個新功能: `rolling update`，也就是**滾動式更新**。也就是說，當樣本Pod的`image`需要更新時，`Deployment`回**逐一**汰換舊的複製品。這樣逐一的汰換過程，可以確保除了正在更新的Pod之外，其他的Pod仍可以提供服務，也就是所謂`Zero Downtime`(零停機更新)，這是`ReplicaSet`做不到的事。

> 所以，一般並不會單純地用`ReplicaSet`來部署服務，更多的是用`Deployment`。

馬上來看一個`Deployment`範例yaml:
```yaml
apiVersion: apps/v1
kind: Deployment # 其實和 ReplicaSet 的 yaml 幾乎一樣，就改 kind 和名字而已 
metadata:
  name: nginx-deploy
spec:
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
        image: nginx
```

* 建立`Deployment`
```bash
$ kubectl apply -f nginx-deploy.yaml
```
> 當然了，你也可以用指令建立Deployment:
```bash
kubectl create deployment nginx-deploy --image nginx --replicas 3
```

* 看看`Deployment`狀況如何
```bash
$ kubectl get deploy nginx-deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           113s
```

* 試試看更新`image`
```bash
$ kubectl set image deployment nginx-deploy nginx=nginx:1.14.2
#注意: 是 <容器名稱>=<新image>，並不是<舊image>=<新image>                   
```

* describe 看看是不是真的更新了
```bash
$ kubectl describe deployment nginx-deploy
...
...(省略)
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
...
...(省略)
```

> 以上只是稍微展示一下'Rolling Update'的效果，後續談到`Application Lifecycle Management`(ALM)時，會再深入討論。(可參考[這裡](12.md))

事實上，當你建立一個`Deployment`，會順便建立`ReplicaSet`，它們的關係如下圖:

![https://ithelp.ithome.com.tw/upload/images/20230926/20161118emE0mmLEST.jpg](https://ithelp.ithome.com.tw/upload/images/20230926/20161118emE0mmLEST.jpg)

利用指令也可以看出這個關係:
```bash
$ kubectl get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           13m

$ kubectl get rs
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-5bbf5d957f   3         3         3       8m42s
nginx-deploy-846d6f46b7   0         0         0       13m 
# 因為更新過一次image，所以AGE較老的上一個版本遺留的
```
以上為ReplicaSet與Deployment的基本介紹。底下來進行練習。

> 由於ReplicaSet與Deployment的概念相似，但Deployment能做的事情更多，所以這次練習將以Deployment為主。

### 練習1: 建立一個Deployment

建立一個Deployment，需求如下:
   * `name`: nginx-deployment
   * `image`: nginx
   * `replicas`: 3
   * `Pod's label`: tier=frontend

### 練習2: 更新剛剛建立的Deployment

更新目標:
   * 更新image為`nginx:1.16.1`

### 練習3: 增加Deployment的replicas

更新目標:
   * `desired number`: 6

### 參考解答

解答請參考[這裡](05-2-ans.md)


### 今日小節
今天從Pod scaling的概念展開討論，帶出了ReplicaSet與Deployment的基本用途與簡單的範例。



