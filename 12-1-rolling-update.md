# *ALM*: Rolling Update & Rollback

在[Day 5](05.md)中，我們稍微展示了`Deployment`的`Rolling Update`的效果，其重點就是「更新」。今天我們來更深入地討論`Rolling Update`與其相對的`Rollback`。

### Update Strategy

當一項應用程式需要更新時，你可以考慮`k8s`中兩種常見的更新策略:
  * **Recreate**: 一次性的刪除所有舊的`Pod`，然後建立新的`Pod`。這種方式會導致服務中斷，也就是所謂的`downtime`。

  * **Rolling Update**: 會一個一個更新舊的`Pod`，直到最後所有的`Pod`都更新完成。這種方式不會導致服務的中斷，也就是所謂的`zero-downtime`。

  以上兩種策略皆可在`Deployment`的`yaml`中`spec.strategy.type`欄位中指定，預設為`RollingUpdate`。

### Rolling Update

假設今天有一個`Deployment`，需求如下:
  * `name`: nginx-deploy
  * `image`: nginx
  * `replicas`: 3
  * `strategy`: `Recreate`

我們先來看看yaml的配置:
```yaml
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
  strategy:
    type:  Recreate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

* 建立後，我們先觀察一下`Pod`的編號:
```bash
kubectl apply -f nginx.yaml
kubectl get pods
# 結果:
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7854ff8877-7fnzm   1/1     Running   0          10m
nginx-7854ff8877-p5s7n   1/1     Running   0          10m
nginx-7854ff8877-p5sgl   1/1     Running   0          10m
```

嘗試更新`Deployment`的`image`，然後觀察`Pod`的更新情況:
```bash
kubectl set image deployment/nginx nginx=nginx:1.15 && kubectl get pods -w
```

可以看到，舊的`Pod`會全部進入`Terminating`狀態，然後新的`Pod`會一個一個被建立(Pending -> ContainerCreating -> Running): 
```text
NAME                     READY   STATUS        RESTARTS   AGE
nginx-7854ff8877-7fnzm   1/1     Terminating   0          11m
nginx-7854ff8877-p5s7n   1/1     Terminating   0          11m
nginx-7854ff8877-p5sgl   1/1     Terminating   0          11m
nginx-7854ff8877-p5sgl   1/1     Terminating   0          11m
nginx-7854ff8877-p5s7n   1/1     Terminating   0          11m
nginx-7854ff8877-7fnzm   1/1     Terminating   0          11m
nginx-7854ff8877-p5sgl   0/1     Terminating   0          11m
nginx-7854ff8877-p5s7n   0/1     Terminating   0          11m
nginx-7854ff8877-7fnzm   0/1     Terminating   0          11m
nginx-86598bfc84-jnnds   0/1     Pending       0          0s
nginx-86598bfc84-jnnds   0/1     Pending       0          1s
nginx-86598bfc84-rhb85   0/1     Pending       0          0s
nginx-86598bfc84-7ppk9   0/1     Pending       0          0s
nginx-86598bfc84-rhb85   0/1     Pending       0          0s
nginx-86598bfc84-7ppk9   0/1     Pending       0          0s
nginx-86598bfc84-jnnds   0/1     ContainerCreating   0          1s
nginx-86598bfc84-rhb85   0/1     ContainerCreating   0          0s
nginx-86598bfc84-7ppk9   0/1     ContainerCreating   0          0s
nginx-7854ff8877-7fnzm   0/1     Terminating         0          11m
nginx-7854ff8877-p5sgl   0/1     Terminating         0          11m
nginx-7854ff8877-7fnzm   0/1     Terminating         0          11m
nginx-7854ff8877-7fnzm   0/1     Terminating         0          11m
nginx-7854ff8877-p5s7n   0/1     Terminating         0          11m
nginx-7854ff8877-p5s7n   0/1     Terminating         0          11m
nginx-7854ff8877-p5s7n   0/1     Terminating         0          11m
nginx-7854ff8877-p5sgl   0/1     Terminating         0          11m
nginx-7854ff8877-p5sgl   0/1     Terminating         0          11m
nginx-86598bfc84-jnnds   0/1     ContainerCreating   0          1s
nginx-86598bfc84-rhb85   0/1     ContainerCreating   0          0s
nginx-86598bfc84-7ppk9   0/1     ContainerCreating   0          0s
nginx-86598bfc84-rhb85   1/1     Running             0          2s
nginx-86598bfc84-7ppk9   1/1     Running             0          2s
nginx-86598bfc84-jnnds   1/1     Running             0          3s
```

> 可以留意一下上面的輸出結果，來和底下的`Rolling Update`做比較。

接著，我們修改一下`yaml`，將`strategy`改為`RollingUpdate`:
```yaml
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
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

> 當然，你也可以直接刪掉`strategy`這個欄位，因為預設就是`RollingUpdate`。

然後，我們直接`apply`這個`yaml`，並觀察`Pod`的更新情況

> 由於`apply`的關係，`yaml`內的配置與現有的`Deployment`會進行比對，發現差異後會根據yaml中的配置進行更新:

```bash
kubectl apply -f nginx.yaml && kubectl get po -w
```

可以看到，不同於`Recreate`，`RollingUpdate`會先建立一個新`Pod`，然後刪除一個舊`Pod`，直到所有`Pod`都更新完成:
```bash
NAME                     READY   STATUS              RESTARTS   AGE
nginx-7854ff8877-rkrkd   0/1     ContainerCreating   0          1s # 建立新的Pod
nginx-86598bfc84-7ppk9   1/1     Running             0          4m6s
nginx-86598bfc84-jnnds   1/1     Running             0          4m7s
nginx-86598bfc84-rhb85   1/1     Running             0          4m6s # 這裡舊的Pod還在跑
nginx-7854ff8877-rkrkd   0/1     ContainerCreating   0          1s
nginx-7854ff8877-rkrkd   1/1     Running             0          2s # 新的Pod建立完成
nginx-86598bfc84-rhb85   1/1     Terminating         0          4m7s # 舊的Pod開始被刪除
nginx-7854ff8877-xq9ht   0/1     Pending             0          0s
nginx-7854ff8877-xq9ht   0/1     Pending             0          0s
nginx-7854ff8877-xq9ht   0/1     ContainerCreating   0          0s
nginx-86598bfc84-rhb85   1/1     Terminating         0          4m7s
nginx-86598bfc84-rhb85   0/1     Terminating         0          4m8s
nginx-7854ff8877-xq9ht   0/1     ContainerCreating   0          1s
nginx-86598bfc84-rhb85   0/1     Terminating         0          4m8s
nginx-7854ff8877-xq9ht   1/1     Running             0          1s
nginx-86598bfc84-rhb85   0/1     Terminating         0          4m8s
nginx-86598bfc84-rhb85   0/1     Terminating         0          4m8s
nginx-86598bfc84-jnnds   1/1     Terminating         0          4m9s
nginx-7854ff8877-lx4rg   0/1     Pending             0          0s
nginx-7854ff8877-lx4rg   0/1     Pending             0          0s
nginx-7854ff8877-lx4rg   0/1     ContainerCreating   0          0s
nginx-86598bfc84-jnnds   1/1     Terminating         0          4m10s
nginx-86598bfc84-jnnds   0/1     Terminating         0          4m10s
nginx-7854ff8877-lx4rg   0/1     ContainerCreating   0          1s
nginx-86598bfc84-jnnds   0/1     Terminating         0          4m10s
nginx-86598bfc84-jnnds   0/1     Terminating         0          4m10s
nginx-86598bfc84-jnnds   0/1     Terminating         0          4m10s
nginx-7854ff8877-lx4rg   1/1     Running             0          1s
```

**補充**
從上面的例子中，我們使用了兩種方式來進行更新:
  1. kubectl apply
  2. kubectl set image

這兩種方式的主要差異在於，`kubectl set image`並且更新後的內容不會反映在`yaml`中，所以在使用時要格外的小心，不要因為貪圖方便而忘記了更新`yaml`。

### Rollback

更新後，可能會發現新版有些問題，想要回到上一個版本。這時候就可以用到`Rollback`:
```bash
kubectl rollout undo deployment/nginx
```

當然你也可以指定回到某個特定的版本:
```bash
kubectl rollout undo deployment/nginx --to-revision=1
```

這時你可能會問了: 「我要怎麼知道要回到哪個版本呢?」，這時候就可以用到`kubectl rollout history`:

> 以上面的nginx為例，總共進行了三次rollout:

 1. 原始版本
 2. 更新到1.15
 3. 回到原始版本
 
```bash
kubectl rollout history deployment/nginx
# 結果:
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

但是，明明總共三次rollout，為什麼只有兩次呢?這是因為REVISION 3實際上是回到REVISION 1(原始版本)，所以在輸出結果中，REVISION 1被「移動」到了REVISION 3。

另外，CHANGE-CAUSE欄位是可以自行指定的，為了清楚的知道每次rollout的原因，在每次更新後可執行以下指令進行標記:
```bash
kubectl annotate deployment/nginx kubernetes.io/change-cause="Rollout to first revision"
```

再次用`kubectl rollout history`查看:
```bash
kubectl rollout history deployment/nginx
# 結果:
REVISION  CHANGE-CAUSE
2         <none>
3         Rollout to first revision
```
> 加上`CHANGE-CAUSE`後，上面關於「移動」的效果就會更明顯了(在練習題會有)。

以上就是`Rolling Update`與`Rollback`的基本操作，以下為練習題。

### 練習1: 進行Rolling Update

創建一個`Deployment`，需求如下:
  * `name`: httpd-deploy
  * `image`: httpd
  * `replicas`: 3

然後，進行一次`Rolling Update`，將`image`更新為`httpd:2.4.39`

### 練習2: 加上CHANGE-CAUSE，並再次進行Rolling Update

將上一次的更新加上`CHANGE-CAUSE`:
  * `CHANGE-CAUSE`: "update to 2.4.39"

並再次進行`Rolling Update`，將`image`更新為`httpd:alpine`

### 練習3: 進行Rollback

### 練習3: 再次加上CHANGE-CAUSE，並進行Rollback

將上一次的更新加上`CHANGE-CAUSE`:
  * `CHANGE-CAUSE`: "update to alpine"

最後，進行`Rollback`，回到第一個版本。