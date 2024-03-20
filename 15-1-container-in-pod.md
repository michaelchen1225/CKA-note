# Containers in Pod

在前一章中，我們使用`command`和`args`來設置`pod`在正式運行的前置作業。當前置作業的複雜姓增加時，例如要**同時**啟動`sidecar`容器，或是要在`pod`**啟動前**先執行較複雜的初始化工作，這時候我們可以考慮用以下兩種手段來實現：
  
  1. **Multi Container Pod**
  2. **Init Containers**

**補充**
sidecar container: 一個附加在主要container旁邊的輔助容器，例如在主要的`web`容器旁邊附加一個`log`容器，用來收集`web`容器的log。

## Multi Container Pod

故名思義，就是一個`pod`中有多個容器。這些容器共享同樣的`network`與`volume`，也屬於**同一個**`Pod`的生命週期，但是各自有自己的`container`和`image`。

常用的應用場景裡如`sidecar container`，底下以一個簡單的範例說明。

**範例情境1**

底下的`yaml`檔案定義了一個`pod`，裡面有兩個`container`:
  1. **main container**: `nginx`容器，用來提供web服務
  2. **sidecar container**: `fluentd`容器，用來收集`nginx`容器的log

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-fluentd
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-log-volume
      mountPath: /var/log/nginx
  - name: fluentd
    image: fluentd
    volumeMounts:
    - name: nginx-log-volume
      mountPath: /var/log/nginx
  volumes:
  - name: nginx-log-volume
    emptyDir: {}

```

> volume的概念會在後面的章節提到，這裡可以先理解成兩個容器共享的「硬碟」。

建立`pod`後，幫它建立一個`service`:
  
```bash
$ kubectl apply -f nginx-fluentd.yaml
$ kubectl expose pod nginx-fluentd --type=NodePort --port=80
$ kubectl get svc nginx-fluentd
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-fluentd   NodePort   10.96.59.112   <none>        80:32134/TCP   46s
```

知道`NodePort`為`32134`後，我們向`nginx`提出請求:

```bash
$ curl localhost:32134
```
回應:
```txt
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

送出請求後，我們可以在`fluentd`容器的「/var/log/nignx/access.log」中看到log:
```bash
$ kubectl exec -it nginx-fluentd -c fluentd -- cat /var/log/nginx/access.log
192.168.0.0 - - [06/Mar/2024:08:40:13 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "-"
```

**選項解釋**
  * -it : 進入互動模式並分配一個tty
  * -c : 指定container

可以看到，我們發送的GET請求被成功的記錄在log中。

## Init Containers

當`Pod`中使用了`initContainers`時，只要這些`initContainers`還沒執行完，其餘的`containers`就不會啟動 ; 除此之外，如果有多個`initContainers`，則它們會按照順序依次執行。

常見的應用場景例如在正式執行前，先執行一些初始化的工作，例如下載資料、設定環境變數等。底下同樣來看一個簡單的範例。

**範例情境2**

底下的`yaml`檔案定義了一個`pod`，裡面有兩個`initContainers`、一個`main container`:
  1. **init-container1**: 輸出訊息"Initializing..."
  2. **init-container2**: 等待5秒
  3. **main container**: `nginx`容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  initContainers:
  - name: init-container1
    image: busybox
    command: ['sh', '-c', 'echo "Initializing..."']
  - name: init-container2
    image: busybox
    command: ['sh', '-c', 'sleep 5']
```

> 其實設定方式和正常的`container`沒有太大差異，只是要在`spec.initContainers`中設定。
  
建立`pod`後，緊接著看`pod`的初始化狀況:
```bash
$ kubuectl apply -f init-demo.yaml && k get po -w
pod/init-demo created
NAME        READY   STATUS     RESTARTS   AGE
init-demo   0/1     Init:0/2   0          0s
init-demo   0/1     Init:0/2   0          1s # 第一個init-container執行完
init-demo   0/1     Init:1/2   0          2s # 第二個init-container開始執行
init-demo   0/1     Init:1/2   0          3s
init-demo   0/1     PodInitializing   0          8s # 過了5秒後，pod開始初始化!
init-demo   1/1     Running           0          9s
```

你可能會好奇`echo`的訊息在哪裡，其實它是在`init-container1`的`stdout`中:
```bash
$ kubectl logs init-demo -c init-container1
Initializing...
```
[上面的參考](https://stackoverflow.com/questions/72077097/why-doesnt-command-line-echo-in-kubernetes-pod-container-show-in-logs)

## 總結
本質上，`multi container pod`和`init containers`都是為了解決`pod`初始化的問題，只是在不同的應用場景下有不同的選擇:

  * `multi container pod`: 適合在`pod`運行時，需要多個容器協同工作的情況
  * `init containers`: 適合在`pod`運行前，需要先執行一些初始化工作的情況。
