## 【Basic Concept】：Rolling Update & Rollback

## 目錄

* [ImagePullPolicy](#imagepullpolicy)

* [Update Strategy](#update-strategy)

* [Rolling Update](#rolling-update)

* [Recreate vs RollingUpdate](#recreate-vs-rollingupdate)

* [Rolling Update 的進階設定](#rolling-update-的進階設定)

* [Rollout history & CHANGE-CAUSE](#rollout-history--change-cause)

* [Rollback](#rollback)

* [Restart](#restart)

在昨天的章節中，我們有提到 Deployment 可以達成 Rolling Update，並且稍微展示了 Update 的效果，也就是更新 image。今天我們來更深入地討論 Rolling update 與 Rollback。不過在此之前，先來看看 K8s 對於 image 的拉取策略。

### ImagePullPolicy

當一個 Pod 需要 image 時，kubelet 會根據 image 的拉取策略(ImagePullPolicy)來判斷該如何處理：

* **IfNotPresent**：只有當 local 沒有該 image 時才會拉取 (預設值)。

* **Always**：每次都會從 remote registry 查詢最新的 image digest。如果 kubelet 在本地快取中有相同 digest 的 image，則使用快取中的 image；反之則會重新拉取 image 來重啟容器。

  > Image 的 digest 是一個**唯一的 hash 值**，可以確保 image 的唯一性(即使 tag 一樣，但 digest 不同，則代表兩個 image 是不同的)。

* **Never**：只使用本地的 image，不會去 Registry 拉取。

ImagePullPolicy 可以在任何用到 image 的地方指定，例如：

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.15
    imagePullPolicy: Never  # 不指定，預設值為 IfNotPresent
```

### Update Strategy

當一個 Deployment 需要更新時，你可以考慮兩種最基本的更新策略：

  * **Recreate**：一次性的刪除所有舊的 Pod ，然後建立新的 Pod。這種方式會導致服務中斷(downtime)。

  * **Rolling Update**: 逐一更新舊的 Pod，一個更新好了才換下一個，直到最後所有的 Pod 都更新完成。這種方式不會導致服務的中斷，因為一個 Pod 在更新時，其他 Pod 還是能被存取到，這就是所謂的 **zero-downtime**。

以上兩種策略皆可在 Deployment 的 yaml「spec.strategy.type」欄位中指定，不指定預設值為 RollingUpdate。

> 因為目前的章節為「Basic Concept」，所以今天先介紹最基礎的更新策略，讓大家初步體驗「Update」的感覺。

> 較為進階的部署策略，例如「**藍綠部署**(blue-green Deployment)」、「**金絲雀部署**(Canary Deployment)」，會在「Workloads & Scheduling」的章節中，也就是 [Day 18](https://ithelp.ithome.com.tw/articles/10348066) 來介紹。

### Rolling Update

Rolling Updates 可以做到以下幾點：

* 更新 image

* 可以 rollback 到之前的版本

* 無論 Update 或是 Rollback，都能達到「Zero-downtime」的效果

我們來實際測試看看 Recreate 與 RollingUpdate 的差異：

### Recreate vs RollingUpdate

今天有一個 Deployment 的需求如下：

  * **name**：nginx-deploy
  * **image**：nginx
  * **replicas**：3
  * **strategy**：Recreate

我們先來看看 yaml 的配置:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
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
```bash
kubectl apply -f nginx.yaml
```

* 建立後，我們先觀察一下 Pod 的「編號」:

```bash
kubectl get pods
```
```text
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7854ff8877-7fnzm   1/1     Running   0          10m
nginx-7854ff8877-p5s7n   1/1     Running   0          10m
nginx-7854ff8877-p5sgl   1/1     Running   0          10m
```

* 更新 Deployment 的 image，然後馬上觀察 Pod 的更新情況：

```bash
kubectl set image deployment/nginx nginx=nginx:1.15 && kubectl get pods -w
```

可以看到，舊的 Pod 會**全部**進入「Terminating」狀態，然後新的 Pod 會一個一個被建立(Pending -> ContainerCreating -> Running): 
```bash
NAME                     READY   STATUS        RESTARTS   AGE
nginx-7854ff8877-7fnzm   1/1     Terminating   0          11m # 舊
nginx-7854ff8877-p5s7n   1/1     Terminating   0          11m # 舊
nginx-7854ff8877-p5sgl   1/1     Terminating   0          11m # 舊
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
nginx-86598bfc84-jnnds   0/1     ContainerCreating   0          1s # 新
nginx-86598bfc84-rhb85   0/1     ContainerCreating   0          0s # 新
nginx-86598bfc84-7ppk9   0/1     ContainerCreating   0          0s # 新
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

> 可以留意一下上面的輸出結果，來和底下的 Rolling Update 做比較。

接著，我們修改一下 yaml ，將 strategy 改為 RollingUpdate：

```yaml
......
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
......
```
> 你也可以直接刪掉 strategy 這個欄位，反正預設就是 RollingUpdate。

然後，我們直接 apply 這個 yaml(這會讓 image 版本從「1.15」更新到「latest」，因為 yaml 裡的 image 沒被更動)，並馬上觀察 Pod 的更新情況：

```bash
kubectl apply -f nginx.yaml && kubectl get po -w
```

可以看到，不同於 Recreate ，RollingUpdate 會先建立一個新 Pod ，然後刪除一個舊 Pod ，直到所有 Pod 都更新完成:

```bash
NAME                     READY   STATUS              RESTARTS   AGE
nginx-7854ff8877-rkrkd   0/1     ContainerCreating   0          1s   # 建立新的Pod
nginx-86598bfc84-7ppk9   1/1     Running             0          4m6s
nginx-86598bfc84-jnnds   1/1     Running             0          4m7s
nginx-86598bfc84-rhb85   1/1     Running             0          4m6s # 舊的Pod還在跑
nginx-7854ff8877-rkrkd   0/1     ContainerCreating   0          1s
nginx-7854ff8877-rkrkd   1/1     Running             0          2s   # 新的Pod建立完成
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

---

> **Tips：更新 image**

從上面的例子中，我們使用了兩種方式來進行更新:
  1. kubectl apply
  2. kubectl set image

這兩種方式的主要差異在於，kubectl set image 並且更新後的內容不會反映在 yaml 中，所以在使用時要格外的小心，有時可能因一時方便而忘記了更新 yaml，造成日後維護上的麻煩 。

***

### Rolling Update 的進階設定

我們還能針對 Rolling Update 的過程進行更多控制：

```yaml
spec:
  minReadySeconds: <N>
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: <X>
      maxUnavailable: <Y>
```

* **minReadySeconds**：在一個 Pod 更新後(Status 為 Ready)，預設上 k8s 會直接更新下一個 Pod，如果這時 Pod 還沒初始化完成就對外開放，部分使用者可能會無法存取服務。透過設定「minReadySeconds」，讓 Pod 在 Ready 後有 n 秒啟動或初始化的時間，然後 k8s 才會繼續更新下一個 template。

* **maxSurge**：X 可以是「整數」或「%。若 X=1，表示最多容許更新過程中存在「desired number + 1」個 Pod 處於 Running 的狀態。

* **maxUnavailable**：Y 可以是「整數」或「%」，但不能同時與 X 設為 0。若 Y=1，表示最多容許更新過程中存在「1」個 Pod 無法提供服務(Status 不是 Ready)。

我們來看幾個例子：

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

> 更新時，k8s 先建立 1 個 Pod，除非新 Pod 的 status 為 Ready，否則不會刪除舊的 Pod。

```yaml
spec:
  minReadySeconds: 5
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
> 更新時，k8s 先建立 1 個 Pod，並至少維持兩個 Pod 是可被存取的(status 為 Ready)。當新 Pod 的 status 為 Ready 後，需等待 5 秒才會更新下一個 Pod。 

```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 20%
      maxUnavailable: 20%
```

> 更新時，k8s 最多可建立 2 個新 Pod，並且至少要維持 8 個 Pod 是可被存取的(status 為 Ready)。

### Rollout history & CHANGE-CAUSE

更新後，我們可以用 kubectl rollout history 來查看更新的歷史紀錄：

> 以上面的 nginx Deployment 為例，總共進行了三次 rollout：

 1. 原始版本
 2. 更新到1.15
 3. 回到原始版本
 
```bash
kubectl rollout history deploy nginx
```
```text
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

但是，明明總共三次rollout，為什麼只看到兩次呢？

這是因為 REVISION 3 實際上是回到 REVISION 1(原始版本)，所以在輸出結果中，REVISION 1被「移動」到了REVISION 3 的位置。

(有關這個「移動」可能現在聽起來有點抽象，不過後面還會多次出現，相信之後會更加清楚。)

另外，CHANGE-CAUSE 欄位是可以自行指定的，為了清楚的知道每次 rollout 的原因，在每次更新後可執行以下指令進行標記：

```bash
kubectl annotate deployment/nginx kubernetes.io/change-cause="Rollout to first revision"
```

再次用查看 rollout history：

```bash
kubectl rollout history deploy nginx
```
```bash
REVISION  CHANGE-CAUSE
2         <none>
3         Rollout to first revision
```


另外，我們也可以單獨針對某個 REVISION，查看到底更新了什麼內容：

```bash
kubectl rollout history deploy nginx --revision=3
```
```text
deployment.apps/nginx with revision #3
Pod Template:
  Labels:       app=nginx
        pod-template-hash=bf5d5cf98
  Annotations:  kubernetes.io/change-cause: Rollout to first revision
  Containers:
   nginx:
    Image:      nginx
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```

### Rollback

在更新之後，可能會發現新版本有些問題，想要回到「上一個」版本。這時候就可以用到 Rollback：

* 回到上一版本，也就是 nginx:1.15
```bash
kubectl rollout undo deployment/nginx
```

* 標記 CHANGE-CAUSE：
```bash
kubectl annotate deployment/nginx kubernetes.io/change-cause="Rollback to 1.15"
```

* 查看 rollout history：
```bash
kubectl rollout history deploy nginx
```
```
3         Rollout to first revision
4         Rollback to 1.15
```

> 可以看到 REVISION 2 消失了，「移動」到了 REVISION 4 的位置，因為兩者其實是相同的版本，只是先後順序不同。而 CHANGE-CAUSE 也從原本的「none」變成了「 Rollback to 1.15」。

你也可以指定回到某個「特定的」版本：

* 我們再進行一次更新：
```bash
kubectl set image deployment/nginx nginx=nginx:l.17
```

* 標記 CHANGE-CAUSE：
```bash
kubectl annotate deployment/nginx kubernetes.io/change-cause="Rollout to 1.17"
```

* 查看 rollout history：
```bash
kubectl rollout history deploy nginx
```
```text
REVISION  CHANGE-CAUSE
3         Rollout to first revision
4         Rollback to 1.15
5         Rollout to 1.17
```
> 這裡就不會有「REVISION 移動」的效果，因為這次是一個全新的 REVISION。


* 「指定」 rollback 到 REVISION 3，此時的版本為 nginx:latest：
```bash
kubectl rollout undo deployment/nginx --to-revision=3
```

* 先不要標記 CHANGE-CAUSE，直接看 rollout history：
```bash
kubectl rollout history deploy nginx
```
```text
REVISION  CHANGE-CAUSE
4         Rollback to 1.15
5         Rollout to 1.17
6         Rollout to first revision
```
> REVISION 3 「移動」到了 REVISION 6 的位置

* 標記 CHANGE-CAUSE：

```bash
kubectl annotate deployment/nginx kubernetes.io/change-cause="Rollback to first revision"
```

### Restart

逐一重啟 Deployment 的所有 Pod：

```bash
kubectl rollout restart deployment/nginx
```


### 今日小結

今天介紹了 Deployment 的 Rolling Update 與 Rollback ，來到到 zero-downtime 的效果，並且可以透過 CHANGE-CAUSE 來標記每次更新的原因，讓管理更加方便。

另外如果忘記 CHANGE-CAUSE 如何標記的話，可以到官網的[Deployment 文件](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)，搜尋「CHANGE-CAUSE」，就可以找到相關的資訊。

-----
**參考資料**

* [Performing a Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)

* [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)