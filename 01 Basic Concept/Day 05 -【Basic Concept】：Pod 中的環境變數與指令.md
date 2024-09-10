# Day 05 -【Basic Concept】：Pod 中的環境變數與指令

### 今日目標

* 設定 Pod 的環境變數

* 設定 Pod 的指令 (command) 與參數 (arguments)

###  Pod 中的環境變數

在許多應用場景中，「環境變數( environment variables )」是一種相當常見設定，而環境變數通常由一對 key-value 組成，例如 USER=root。


那麼在 Pod 的 yaml 中可以在「spec.containers[].env[]」底下定義環境變數：

```yaml
# env-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - command: ["sleep", "300"]
    env:   # environment variables
    - name: USER     # key
      value: michael # value
    image: busybox
    name: busybox
```
```bash
kubectl apply -f env-demo.yaml
```

建立 Pod 後，來測試看看環境變數是否有設定成功：

```bash
kubectl exec -it env-demo -- /bin/sh
```

* 進入容器後，嘗試取得環境變數的值：
```bash
echo $USER
```
輸出：
```text
michael
```

> 用 kubectl run 建立 Pod，使用「--env」設定環境變數：

```bash
kubectl run env-demo-2 --image busybox --env="USER=Mike" --command -- sleep 300
```

如果要定義多個環境變數，就在「spec.containers.env[]」中加入多個 key-value即可，例如:
```yaml
......
containers:
- command:
  - sleep
  - "300"
  env: # 環境變數
  - name: USER
    value: michael
  - name: PASSWORD
    value: 123456
......
```

> 用 kubectl run 建立 Pod，用多個「--env」設定多個環境變數：

```bash
kubectl run env-demo-3 --image busybox --env="USER=Mike" --env="PASSWORD=123456" --command -- sleep 300
```

不過，當需要的環境變數開始變多時這樣設定非常沒有效率，且不易閱讀。這個問題我們暫且留到後續章節，介紹 ConfigMap 時再來解決。

### 將 Pod 的資訊當作環境變數

當 Pod 跑起來後，我們可以將一些與 Pod 相關的變數當作環境變數，例如 Pod 的名稱、IP 等等，例如：

```yaml
# env-from-pod-info.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-from-pod-info
spec:
  containers:
    - name: test
      image: busybox
      command: [ "sh", "-c", "echo $MY_NODE_NAME $MY_POD_NAME $MY_POD_NAMESPACE $MY_POD_IP" ]
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName  
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
```
```bash
kubectl apply -f env-from-pod-info.yaml
```

> 基本上就是加入「valueFrom」，然後指定「fieldRef.fieldPath」，範例中我們依序取得：

* Pod 所在的 Node 名稱
* Pod 的名稱
* Pod 所在的 Namespace
* Pod 的 IP 

> 如果不知道特定欄位怎麼表示，可以這樣看：

```bash
kubectl get pod <pod_name> -o yaml
```
建立 Pod 後，查看 logs：

```bash
kubectl logs env-from-pod-info
```
輸出：
```text
node01 env-from-pod-info default 192.168.1.9
```


### Pod 中的指令 (command) 與參數 (arguments)

如果用用指令建立一個 busybox 的 Pod ，且不帶任何參數，看看會發生什麼事：

```bash
kubectl run busybox --image busybox
```

busybox 在最初執行時會產生一個 shell 等待 input，但我們卻沒提供任何指令(command)或參數(arguments)，Pod 就會以為沒有任務了，所以立即結束。

> 又因為 Pod 的 `restartPolicy` 預設是 Always，所以它會不斷重啟，然後再次結束：

```bash
kubectl get po busybox
```
```text
NAME      READY   STATUS             RESTARTS      AGE
busybox   0/1     CrashLoopBackOff   6 (69s ago)   6m56s
```

因此我們透過給 busybox 一個指令，讓它不會立即結束：

```bash
kubectl run busybox --image busybox --command -- sleep 3600
```

> 當 sleep 3600 結束後沒有任和輸入， Pod 還是會結束。

如果想要與 Pod 中的容器使用「終端」進行互動，可以使用：
```bash
kubectl exec -it busybox -- /bin/sh
```

> 更嚴謹的說，是與 Pod 中的「第一個 container 」進行互動 (因為 Pod 可以有多個 container，但是這裡只有一個)，如果要指定某個 container，需要使用 -c 選項指定：

```bash
kubectl exec -it busybox -c <container_name> -- /bin/sh
```
輸入 exit 後就可以離開。

---

**kubectl exec 選項解釋**

* -i: 進入互動模式
* -t: 分配一個tty(終端機)

***

底下我們就來談談 command 與 arguments 在 Pod 中的設定。不過在此之前，我們來看看 Dockerfile 中的是怎麼設定的。

### Command & Arguments in Dockerfile

我們知道 Linux 指令的基本格式是：
```bash
<command> [-option] <arguments>
```

舉例而言，「sleep 300」中，「sleep」是 command，「300」是 arguments，而這行指令沒有 option。

那麼，在 Pod 中真正在執行指令的是 container，container 由 image 產生，而 image 是由 Dockerfile 建立的，例如：

* 我們想讓一個 container 預設可以執行「echo 1」，可以在 Dockerfile 中這樣寫:

```Dockerfile
FROM alpine
CMD ["echo", "1"]
```

**注意**：不要寫成CMD ["echo 1"]

筆者已經將上面的 Dockerfile 建立成 image，並命名為 my-echo，版本為 v1。

> 如果沒有安裝 docker，先把 docker 安裝好：
```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli
```

把器跑起來並命名為 echo-1：
```bash
docker run --name echo-1 michaelchenn1225/my-echo:v1
```
輸出：
```
1
```

但是如果想要讓容器輸出「hello」，就必須把「echo hello」加在「ctr run」最後面，取代原本的「echo 1」:
```bash
docker run --name echo-hello michaelchenn1225/my-echo:v1 echo hello
```
輸出：
```text
hello
```

所以，如果確定我們想讓容器完成的就是「echo XXX」，我們可以將 Dockefile 改成：

```Dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["2"] 
```
> ENTRYPOINT 對應到的就是 Linux 指令的 command，而 CMD 對應到的就是 arguments。

這時「echo 2」就會是預設的指令，同時也能以更方便、直覺的方法，讓我們改成下達「echo hello」:

* 先照預設的指令輸出「2」:
```bash
docker run --name echo-2 michaelchenn1225/my-echo:v2
```
輸出：
```text
2
```

* 建立一個容器，取名為 echo-hello-v2，改成輸出「hello-v2」:
```bash
docker run --name echo-hello-v2 michaelchenn1225/my-echo:v2 hello-v2
```
輸出：
```text
hello-v2
```

最後，如果我我真的不想執行「echo XXX」，想改成可輸出環境變數的指令「env」，可以這樣下指令：
```bash
docker run --name my-sleeper --entrypoint="env" michaelchenn1225/my-echo:v2 
```
輸出：
```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=1ffc3b13a131
HOME=/root
```

### Command & Arguments in Pod

了解了 Dockerfile 中的 command 與 arguments 後，我們來看看 Pod 中的設定：
  
假如我想要在 Pod 中使用剛才的 image 來執行「echo 2」，可以這樣編寫 yaml 檔：

```yaml
# echo-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo-2
spec:
  containers:
  - name: echo-2
    image: michaelchenn1225/my-echo:v2
```
```bash
kubectl apply -f echo-2.yaml
```

建立 Pod 後，等待 Pod 執行完畢後，檢查 Pod 的 logs：
```bash
kubectl logs echo-2
```
輸出：
```text
2
```

假果想要將輸出改為「hello-v2」，使用「args」來設定給 echo 的 arguments：
```yaml
# echo-hello.yaml
apiVersion: v1
kind: Pod
metadata:
  name: echo-hello
spec:
  containers:
  - name: echo-hello
    image: michaelchenn1225/my-echo:v2
    args: ["hello-v2"]
```
```bash
kubectl apply -f echo-hello.yaml
```

同樣查看一下 logs：
```bash
kubectl logs echo-hello
```
輸出：
```text
hello-v2
```

那如果我想改成執行「cat /etc/os-release」，可以這樣寫：
```yaml
# my-cat.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-cat
spec:
  containers:
  - name: my-cat
    image: michaelchenn1225/my-echo:v2
    command: ["cat"]
    args: ["/etc/os-release"]
```
```bash
kubectl apply -f my-cat.yaml
```

查看 logs：
```bash
kubectl logs my-cat
```
輸出：
```text
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.20.2
PRETTY_NAME="Alpine Linux v3.20"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
```

> 或者也可以這樣寫：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-sleep
spec:
  containers:
  - name: my-sleep
    image: my-sleep
    command: ["cat", "/etc/os-release"] 
```
**注意**：要記得用雙引號隔開指令與參數，不要寫成 command: ["cat /etc/os-release"]，因為這樣會被解讀成一個叫做「cat /etc/os-release」的指令！

測試了這麼多例子後，可以看出 Dockerfile 與 yaml 的對應關係如下：

Dockerfile | yaml
--- | ---
ENTRYPOINT | command
CMD | args

也就是在 Pod 的 yaml 中，command 欄位會取代 image 的 `ENTRYPOINT`，而 args 會取代原先 image 的 `CMD`。

---
> **Tips**：執行長指令或多重指令

如果要執行一個長指令，在 yaml 中一直用雙引號格開會相當麻煩，因此可以使用以下小技巧：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: long-command
spec:
  containers:
  - name: long-command
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo hello; sleep 10; done"]
```

有的時候我們會想執行多個指令，例如先輸出 hello，再等待 300 秒，可以這樣寫：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-command
spec:
  containers:
  - name: multi-command
    image: busybox
    command: ["/bin/sh", "-c", "echo hello ; sleep 300"]
```
***


### 今日小結

今天透過 Dockerfile 與 yaml 的比較，來介紹 Pod 中的 command 與 arguments 的設定方式。總之，使用「/bin/sh -c」是相當常用且方便的技巧，可以讓我們在 yaml 中更方便的設定指令。

底下我們用一個簡單的實作來綜合運用一下今天的內容：

*建立一個 Pod，包含兩個容器*：

* 容器：
  * 取名為 echo-user，使用 busybox 作為image ，定義一個環境變數為「myUser=u1」，先等待 10 秒後再將環境變數 u1。
  * 取名為 echo-user-2，使用 busybox 作為image ，定義一個環境變數為「myUser=u2」，每隔 1 秒將 u2 輸出。

* 建立 yaml 樣本：
```bash
kubectl run echo-user --image busybox --env="myUser=u1" --dry-run=client -o yaml --command -- sleep 10 > env-demo.yaml
```

* 修改 yaml，加入第二個容器

```yaml
# env-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: echo-user
  name: echo-user
spec:
  containers:
  - command: ["sh", "-c", "sleep 10; echo $myUser"] # change
    env:
    - name: myUser
      value: u1  
    image: busybox
    name: echo-user
    resources: {}
  - command: ["sh", "-c", "while true; do echo $myUser; sleep 1; done"] # add the second container
    env:
    - name: myUser
      value: u2
    image: busybox
    name: echo-user-2
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}        
```
```bash
kubectl apply -f env-demo.yaml
```

* 等 Pod 跑起來 10 秒後，查看第一個容器的 logs：
```bash
kubectl logs echo-user -c echo-user
```
輸出：
```text
u1
```

* 查看第二個容器的 logs：
```bash
kubectl logs echo-user -c echo-user-2 
```
輸出：
```text
u2
u2
u2
.......
```

-----
**參考資料**

* [Define Environment Variables for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)

* [Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)

* [CrashLoopBackOff status using busybox images](https://www.reddit.com/r/kubernetes/comments/jp2ia9/crashloopbackoff_status_using_busybox_images/):

* [Kubernetes: Command and Arguments in Pod](https://yuminlee2.medium.com/kubernetes-command-and-arguments-in-pod-c3f1be61ba1a)