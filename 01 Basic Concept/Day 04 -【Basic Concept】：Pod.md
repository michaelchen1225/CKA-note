### 今日目標

* 了解如何建立 Pod
  * 使用 yaml 建立
  * 使用 kubectl 建立
  * 快速產生 yaml 樣本

* 關於 Pod 的 kubectl 基本操作

* Pod 的基本除錯 --- 使用 kubectl describe 和 kubectl logs

* Multi-Container Pod
  * Init Container
  * Sidecar Container

### 如何建立 Pod？

「Pod」是 K8s 的基本執行單位，可以用以下兩種方法來建立：

   * 方法1：使用 **yaml** 檔案

   * 方法2：使用 **kubectl** 指令

---
> **Tips：yaml**

如果還不太清楚 yaml 的格式或寫法，可以參考：
  
  * [YAML 基本使用筆記](https://www.letswrite.tw/yaml-basic/)

  * [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
***

### 方法1：使用 yaml 檔案建立 Pod

直接來看一個 k8s 官網上的[範例](https://kubernetes.io/docs/concepts/workloads/pods/#using-pods)：

```bash
wget https://k8s.io/examples/pods/simple-pod.yaml
cat simple-pod.yaml
```
```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

我們先從大標題，也就是 apiVersion、kind、metadata、spec 開始看起：

* **apiVersion**：

  為了方便管理，k8s 將不同的物件資源用 API 版本來分類，而 Pod 屬於「v1」這個 API 版本之下。

* **kind**：
   
  在 API 版本為 v1 的情況下，指定要創建的資源種類。範例是 Pod。

* **metadata**：
  
  描述這個物件的訊息，例如: name (名稱)、labels (標籤)、anottations (註釋) 等等。範例中這個 Pod 被命名為「nginx」。

* **spec**：
  
  定義了這個物件的規格，範例中的 spec.containers 欄位之下，定義了 Pod 中的容器該有哪些配置：
   
   1. name：容器取名為「nginx」

   2. image：這裡使用「nginx:1.14.2」

   3. ports：容器會開放 port 80


寫完 Pod 的 yaml 後，用以下的方式建立就能建立 Pod：

```bash
kubectl apply -f simple-pod.yaml
```
或是:
```bash
kubectl create -f simple-pod.yaml
```

查看 nginx Pod 的狀態，可以看到是 Running 狀態：
```bash
kubectl get pod nginx
```
```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          72s
```

---
> **Tips：apply vs create**

* kubectl apply -f：

  用來創建新的資源，也可以用來**更新**現有的資源。如果指定 yaml 中的資源已經存在，kubectl apply 會根據 yaml 的內容對現有資源進行更新，而不是將其視為一個全新的資源。

* kubectl create -f：

  用於創建新的資源，如果指定 yaml 中的資源已經存在，則會出現錯誤，而不像 apply 那樣會更新資源。

  總而言之 kubectl apply -f 是一個*更靈活*的指令，比較推薦使用。
***

### 方法2: 使用 kubectl 指令建立 Pod

用 kubectl run 指令來產生 Pod，指令格式如下：

```bash
kubectl run <pod-name> --image <image> --port <port> 
```

如果要建立一個與「方法1」範例中一模一樣的 Pod，那麼實際執行的指令就是：

```bash
kubectl run nginx --image nginx:1.14.2 --port 80
```

你可能會問，*那我幹嘛辛苦地寫一個 yaml，不是直接用指令就好了？*

指令提供的是一個快速、便捷的方式來創建資源，但一些較為細節的設定還是需要透過 yaml 來配置。另外使用 yaml 還有這些好處：

   * **可讀性和維護性**：
     yaml 在複雜的應用之中，因為其書寫的結構方式，大大提升了閱讀與維護的速度。
 
   * **版本控制**：
     yaml 可以輕鬆的儲存在版控系統中(例如 Git)，適合開發團隊的合作。

> **CKA Tips**：在考試中為了節省時間，能用指令的地方就用指令，除非指令無法滿足需求在使用 yaml。

既然指令的選項無法滿足需求時，還是需要用到 yaml，難道這時就要自己重頭到尾寫一份 yaml 嗎？

其實還是可以使用指令的方式，先產生基本的 yaml **樣本**，隨後再利用文字編輯器(例如 vim、nano)進行符合要求的修改 :

* 生成 yaml 的樣本:

```bash
kubectl run <pod-name> --image=<image> --dry-run=client -o yaml > <yaml-name>.yaml
```

如果仔細看，其實就是原本的 kubectl run 指令後面加上兩個選項 *--dry-run=client* 與 *-o yaml*，再將輸出結果丟到一個 yaml 檔案裡罷了。這兩個選項各別表示：

   * --dry-run=client：

     讓命令執行後，只會檢查命令的有效性，而不會真正地創建 Pod

   * -o yaml：

     指定輸出格式，這裡指定輸出為 yaml
     
舉例來說，我們無法透過 kubectl run 建立一個包含多個容器的 Pod，不過我們可以先生成單一容器的 yaml 檔，再進行修改：

```bash
kubectl run multi-container-pod --image=nginx --dry-run=client -o yaml > multi-container-pod.yaml
```

修改 yaml ，加入第二個容器：

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-container-pod
  name: multi-container-pod
spec:
  containers:
  - image: nginx
    name: nginx # 修改容器名稱
    resources: {}
  - image: busybox #新增的容器
    name: busybox
    command: ["sleep", "300"]
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
```bash
kubectl apply -f multi-container-pod.yaml
```

這樣就可以在不用從頭開始寫 yaml 的情況下，快速地建立一個多容器的 Pod：

```bash
kubectl get pod multi-container-pod 
```
```text
NAME                  READY   STATUS    RESTARTS   AGE
multi-container-pod   2/2     Running   0          32s
```
> 可以看到 READY 欄位是 2/2，表示這個 Pod 中有兩個容器，且都是 Ready 狀態。

### 其他Pod的基本操作指令

建立 Pod 之後，你可能會問：我要怎麼知道 Pod 有沒有正常運作？哪裡可以查看關於這個 Pod 的詳細資料？如何刪除 Pod？有沒有其他與 Pod 相關的指令可以使用？

列出目前所有 Pod：
```bash
kubeclt get pod
```
> 這樣講其實並不準確，不過因為還沒講到 namespace 的概念，所以先暫時這樣理解

持續的列出 Pod 的狀態：
```bash
kubectl get pod -w
```

查看 Pod 的詳細資訊，例如 Pod 的 status、events、容器名稱等等：
```bash
kubectl describe pod <pod-name>
```

查看 Pod 中「容器的 log 」：
```bash
kubectl logs <pod-name>
```

持續的輸出容器的 log：
```bash
kubectl logs -f <pod-name>
```

若 Pod 中的容器不只一個，可以用「-c」來指定容器：
```bash
kubectl logs <pod-name> -c <container-name>
```

> **範例**：在剛才建立的 multi-container-pod 中，指定查看 nginx 容器的 log：

```bash
kubectl logs multi-container-pod -c nginx
```

刪除 Pod (可以一次刪除多個，pod-name 用空格隔開)：
```bash
kubectl delete pod <pod-name> <pod-name> ...
```
> **範例**：刪除剛剛建立的 nginx Pod :

```bash
kubectl delete pod nginx
```

強制性的刪除 Pod：
```bash
kubectl delete pod --force --grace-period=0
```
> **CKA Tips**：有時候刪除 Pod 的時候會一直卡在 Terminating 的狀態，就可以用上面的方式強制刪除來節省時間。

刪除使用 yaml 檔創建的 Pod：
```bash
kubectl delete -f <yaml-path>
```

> **範例**：刪除用 yaml 建立的 multi-container-pod

```bash
kubectl delete -f multi-container-pod.yaml --force
```

建立 Pod 並執行在容器中執行指令：
```bash
kubectl run <pod-name> --image <image> --command -- <command> <arg1> <arg2> ... <argN>
```

> **範例**：創建一個 busybox pod，並執行指令 sleep 300

```bash
kubectl run busybox --image busybox --command -- sleep 300
```

建立 Pod 並使用預設指令，但使用不同參數：
```bash
kubectl run <pod-name> --image <image> -- <arg1> <arg2> ... <argN>
```

(關於 Pod 中的指令與參數，會在之後介紹)

在執行中的 Pod 裡執行指令：
```bash
kubectl exec <pod-name> -- <command> <arg1> <arg2> ... <argN>
```
> **範例**：在剛剛建立的 busybox Pod 裡執行指令 echo hello

```bash
kubectl exec busybox -- echo hello
```

使用終端與 Pod 中的容器進行互動：
```bash
kubectl exec -it <pod-name> -- /bin/sh
```
> **範例**：使用 sh shell 與剛剛建立的 busybox 互動

```bash
kubectl exec -it busybox -- /bin/sh
# 使用 exit 離開
```

更新 Pod 的 image：
```bash
kubectl set image pod <pod-name> <container-name>=<new-image>
```
> 例如有一個 Pod 名為 webapp，裡面的容器名稱為 web，要更新 image 為 nginx:1.15：

```bash
kubectl set image pod webapp web=nginx:1.15
```

搭配 jsonpath 來取得 Pod 的特定資訊:
```bash
kubectl get pod -o jsonpath="{<json path expression>}"
```
> **範例**: 抓出 busybox Pod 的 IP：

```bash
kubectl get pod busybox -o jsonpath='{.status.podIP}'
```

---
> **Tips：jsonpath**

有關 jsonpath 的應用可參考[附錄](https://github.com/michaelchen1225/CKA-note/blob/main/%E9%99%84%E9%8C%84/%E9%99%84%E9%8C%841-jsonpath.md)。
***

### Pod 的基本除錯

在上面的指令中，kubectl describe 和 kubectl logs 是相當基本且重要的除錯指令，兩者的差別主要是：

* kubectl describe：以 cluster 的角度來觀察 Pod 與容器的狀態。

* kubectl logs：以 container 的角度書出 Pod 中「容器的 log」

這樣說可能比較抽象，我們來看幾個例子，順便練習一下有關 Pod 的操作 :

* 建立一個 nginx 的 Pod，不過故意打錯 image 的名稱，讓它無法正常運作:
```bash
kubectl run nginx --image nginxxx
```
* 先以 cluster 的角度來觀察這個 Pod 的狀況:

```bash 
kubectl describe pod nginx
```
```text
......
Containers:
  nginx:
    Container ID:   
    Image:          nginxxx
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ErrImagePull
    Ready:          False
    Restart Count:  0
...(省略)...
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  7s    default-scheduler  Successfully assigned default/nginx to controlplane
  Normal   Pulling    6s    kubelet            Pulling image "nginxxx"
  Warning  Failed     6s    kubelet            Failed to pull image "nginxxx": failed to pull and unpack image "docker.io/library/nginxxx:latest": failed to resolve reference "docker.io/library/nginxxx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     6s    kubelet            Error: ErrImagePull
  Normal   BackOff    5s    kubelet            Back-off pulling image "nginxxx"
  Warning  Failed     5s    kubelet            Error: ImagePullBackOff
```

**解讀**

kubectl describe 的輸出結果中，較為重要的是輸出訊息分別是 **state** 和 **events**：

  * 從 Containers 欄位底下的 state 看到，這個 Pod 的狀態是「Waiting」，且 Reason 是「ErrImagePull」，這表示 Pod 無法正常運作的原因是因為無法拉取 image。

  * Events 欄位則是列出了 Pod 的事件，可以看到 Pod 的最新事件的 Type 是「Warning」，且 Reason 是「Failed」，從 Message 中可以看到更詳細的錯誤訊息。

既然 Pod 還沒起來，那自然就不會有 container 的 log，所以當我們嘗試使用 kubectl logs 指令時，會得到以下的錯誤訊息:

```bash
kubectl logs nginx
```
錯誤訊息：
```text
Error from server (BadRequest): container "nginx" in pod "nginx" is waiting to start: trying and failing to pull image
```
> container 正在等待 Pod 的啟動，因此無法取得 log

* 既然如此，我們先將 Pod 的 image 名稱改正：
```bash
kubectl set image pod nginx nginx=nginx
```

* 再次用 kubectl describe 查看 Pod 的狀況:
```bash
kubectl describe pod nginx
```
```text
......
Containers:
  nginx:
    Container ID:   containerd://bd4b11a1f23d2b886b30d11bc35e5d0da9f8ecbd11dfd6722d8eda3f6cdfb948
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:0463a96ac74b84a8a1b27f3d1f4ae5d1a70ea823219394e131f5bf3536674419
    Port:           <none>
    Host Port:      <none>
    State:          Running #成功執行!
      Started:      Sat, 20 Apr 2024 09:41:57 +0000
    Ready:          True
    Restart Count:  0
......
```

* 使用 kubectl logs 查看 Pod 的 log:
```bash
kubectl logs nginx
```
```text
......
2024/04/20 09:41:57 [notice] 1#1: start worker process 78
2024/04/20 09:41:57 [notice] 1#1: start worker process 79
2024/04/20 09:41:57 [notice] 1#1: start worker process 80
......
```
> 因為 Pod 已經成功執行，所以就可以看到 container 的 log

我們再來看另一個例子:

* 建立一個 nginx 的 Pod，並將其命名為 web:
```bash
kubectl run web --image nginx
```

* 以cluster的角度來觀察這個 Pod 的狀況:
```bash
kubectl describe pod web
```
```text
......
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  27s   default-scheduler  Successfully assigned default/web to controlplane
  Normal  Pulling    26s   kubelet            Pulling image "nginx"
  Normal  Pulled     26s   kubelet            Successfully pulled image "nginx" in 173ms (173ms including waiting)
  Normal  Created    26s   kubelet            Created container web
  Normal  Started    26s   kubelet            Started container web # 開始執行容器
```

* 將 Pod 的 IP 儲存到變數中：
```bash
POD_IP=$(kubectl get pod web -o jsonpath="{.status.podIP}")
```

* 然後curl到這個 IP：
```bash
curl ${POD_IP}
```

* 可以看到 Pod 成功的回應了一個初始的 nginx 頁面，而這個回應可以在 kubectl logs 中看到：
```bash
kubectl logs web
```
```text
......
2024/08/22 08:50:04 [notice] 1#1: start worker process 28
192.168.0.0 - - [22/Aug/2024:08:50:45 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "-"
......
```

> 可以看到 Pod 回應了一個 GET 請求，且狀態碼為 200，這表示請求成功

如果我們用 describe 指令查看 Pod 的 Events，Events中並沒有看到這個請求，原因是「回應 curl」並不屬於 cluster 層級的事件，而是屬於 container 層級的事件。這樣是不是能夠比較了解「以 cluster 的角度」和「以 container 的角度」的差異呢？

### Multi-Container Pod

前面提到 Pod 中可以有多個容器，而這些容器可以分為兩種常見的應用類型：Init Container 和 Sidecar Container。

### Init Container

在 Pod 中「最先開始」執行，如果 initContainers 還沒執行完，其餘的 containers 就不會啟動。

> 如果有多個 initContainers，則會按照順序來執行。

常見的應用場景例如在主容器執行前，先進一些初始化的工作，例如下載資料、設定環境變數等。底下來看一個簡單的實作：

* 定義一個叫做「init-demo」的 Pod，裡面有兩個 initContainers、一個 main container：

  * initContainers：
    * init-container1：使用 busybox 作為 image，並輸出訊息「Initializing...」
    * init-container2:：使用 busybox 作為 image，等待 5 秒
  
  * main container：
    * nginx：使用 nginx 作為 image，並開放 port 80。

首先生成一個 yaml 範本：
```bash
kubectl run init-demo --image=busybox --dry-run=client -o yaml --command -- sh -c "echo 'Initializing...'" > init-demo.yaml
```

修改 yaml：

```yaml
# init-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: init-demo
  name: init-demo
spec:
  initContainers: # containers -> initContainers
  - command:
    - sh
    - -c
    - echo 'Initializing...'
    image: busybox
    name: init-container1  # init-demo -> init-container1
    resources: {}
  - command: ["sh", "-c", "sleep 5"] # add init-container2
    image: busybox 
    name: init-container2
  containers: # add main container
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
> 其實 initContainers 的 yaml 寫法跟一般容器一模一樣，只是要放在「spec.initContainers」底下。

建立 Pod 後，馬上查看 Pod 的初始化狀況：
```bash
kubectl apply -f init-demo.yaml && kubectl get po init-demo -w
```
```text
pod/init-demo created
NAME        READY   STATUS     RESTARTS   AGE
init-demo   0/1     Init:0/2   0          0s
init-demo   0/1     Init:0/2   0          1s # 第一個init-container執行完
init-demo   0/1     Init:1/2   0          2s # 第二個init-container開始執行
init-demo   0/1     Init:1/2   0          3s
init-demo   0/1     PodInitializing   0          8s # 過了5秒後，main container 才開始初始化
init-demo   1/1     Running           0          9s
```

你可能會好奇 echo 的訊息在哪裡，其實它是在「init-container1」的 stdout 中：

```bash
kubectl logs init-demo -c init-container1
```
輸出：
```text
Initializing...
```

**Sidecar Container**

Sidecar container 其實就是普通的容器，會與主容器同時運作並扮演著輔助的角色，例如在主要的 web 容器旁邊附加一個 log 容器，用來收集 web 容器的log，底下我們來實作看看：

* 定義一個叫做 sidecar-demo 的 Pod，包含底下兩個容器：
  * 主容器名為「main-container」，image 為 busybox，使用 while 迴圈每隔一秒將日期輸出到 /var/log/date.log
  * sidecar 容器名為 sidecar-container，image 為 busybox，使用 tail -f 持續輸出 /logs/date.log

* 透過一個叫做 shared-logs 的 volume 來讓兩個容器共享 date.log，掛載點為：
  * main-container：/var/log
  * sidecar-container：/logs

---
**補充說明**

> volume 的概念會在後面的 [Day 13](https://ithelp.ithome.com.tw/articles/10347182) 介紹，這裡可以先理解成兩個容器共享一個叫做 shared-logs 的「硬碟」，分別掛載到 :

* main-container 的 /var/log
* sidecar-container 的 /logs

當 main-container 每隔 1 秒輸出一次日期到 /var/log/date.log 後，因為「shared-logs 硬碟」的關係，sidecar-container 可以在 /logs 底下找到 date.log 並輸出。
***

首先，生成 yaml 樣本：
```bash
kubectl run sidecar-demo --image busybox --dry-run=client -o yaml --command -- sh -c "while true; do date >> /var/log/date.log; sleep 1; done" > sidecar-demo.yaml
```

修改 yaml：
```yaml
# sidecar-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sidecar-demo
  name: sidecar-demo
spec:
  volumes:     # add volume
    - name: shared-logs
      emptyDir: {}
  containers:
  - command:
    - sh
    - -c
    - while true; do date >> /var/log/date.log; sleep 1; done
    image: busybox
    name: main-container    # sidecar-demo -> main-container
    volumeMounts:           # add volume mount
    - name: shared-logs     # Volume name(spec.volumes.name)
      mountPath: /var/log   # Mount path
  - command: ["sh", "-c", "tail -f /logs/date.log"] # add sidecar container
    image: busybox
    name: sidecar-container
    volumeMounts:           # add volume mount
    - name: shared-logs
      mountPath: /logs
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

建立 Pod：
```bash
kubectl apply -f sidecar-demo.yaml
```

等待兩個 Pod 都跑起來後，查看 sidecar-container 的 log：
```bash
kubectl logs -f sidecar-demo -c sidecar-container 
```
> 可以看到 sidecar-container 不斷地輸出日期！

### 今日小結

今天介紹了 Pod 的一些基本操作，建議讀者將每個指令與操作都實際操作一遍，熟悉一遍後不妨試試看能不能自己還原出 initContainer 與 sidecar 的實作，如果可以的話相信對今天內容就有一定的掌握了！

另外，如果對 kubectl 的指令不熟的話，可以到官網查詢，或是在指令後加上 --help 或 -h 來查看該指令的使用方式。

-----
**參考資料**
* [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
* [kubectl run](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
* [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
* [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
* [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
* [Why doesn't command line echo in Kubernetes Pod container show in logs](https://stackoverflow.com/questions/72077097/why-doesnt-command-line-echo-in-kubernetes-pod-container-show-in-logs)
* [CKAD Mod 1: Understand Multi-Container Pod Design Patterns](https://rx-m.com/lesson/ckad-understand-multi-container-pod-design-patterns/)