# "ALM": Command & Arguments

如果用用指令建立一個`busybox`的`Pod`，且不帶任何參數，看看會發生什麼事:
```bash
kubectl run busybox --image busybox
```

[參考](https://www.reddit.com/r/kubernetes/comments/jp2ia9/crashloopbackoff_status_using_busybox_images/):

`busybox`在最初運行時會產生一個`shell`等待輸入，但我們卻沒提供任何指令(command)或參數(arguments)，它就會「以為」沒有任務了，所以立即結束。

> 又因為`Pod`的`restartPolicy`預設是`Always`，所以它會不斷重啟，然後再次結束: 

```bash
$ kubectl get po
NAME      READY   STATUS             RESTARTS      AGE
busybox   0/1     CrashLoopBackOff   6 (69s ago)   6m56s
```

在[Day04](04-1.md)的練習題中，我們透過以下方式給`busybox`一個指令，讓它不會立即結束:

```bash
kubectl run busybox --image busybox --command -- sleep 3600
```

這樣`Pod`就會一直處於`sleep`狀態，不會立即結束。
> 但是，當`sleep 3600`結束後沒有任和輸入，`Pod`還是會結束。

那如果想要進到`Pod`內部下達指令，可以使用:
```basj
kubectl exec -it busybox -- /bin/sh
```
> 更嚴謹的說，是進入`Pod`中的「第一個`container`」 (因為`Pod`可以有多個`container`，但是這裡只有一個)，如果要指定某個`container`，可以使用`-c`參數:

```bash
kubectl exec -it busybox -c <container_name> -- /bin/sh
```
輸入`exit`後就可以離開。

**選項解釋**
* -i: 進入互動模式
* -t: 分配一個tty(終端機)

底下我們就來談談`command`與`arguments`在`k8s`中的設定。不過在此之前，我們得先看看`Docker`中的是怎麼設定的。

## Command & Arguments in Docker

[參考](https://yuminlee2.medium.com/kubernetes-command-and-arguments-in-pod-c3f1be61ba1a)

我們知道在指令的基本格式是:
```bash
<command> [-option] <arguments>
```

而`pod`中其實真正執行指令的是`container`，以`Docker`為例，`command`與`arguments`是在'`Dockerfile`'中設定的:

假如我們想要在容器中執行`sleep 3600`，可以在`Dockerfile`中這樣寫:

```Dockerfile
    FROM alpine
    CMD ["sleep", "3600"] # 把command和arguments寫在一起
```
**注意**: 不要寫成CMD ["sleep 3600"]


寫完後，我們將'Dockerfile'建立成`image`，名為「my-sleep」:
```bash
docker build -t my-sleep .
```

執行容器後就會休眠60分鐘:
```bash
docker run my-sleep
```

但是如果今天想要縮短休眠時間為1分鐘`，就必須加上「sleep 60」，取代原本的「sleep 3600」:
```bash
    docker run my-sleep sleep 60
```

所以，如果我們確定會下達的一定是「sleep XXX」，我們可以將`Dockefile`改成:

```Dockerfile
FROM alpine
ENTRYPOINT ["sleep"] # command
CMD ["3600"] # arguments
```

這時「sleep 3600」就會是預設的指令，同時也以更方便、直覺的方法讓我們改成下達「sleep 60」:
```bash
docker run my-sleep 60
```

最後，如果我我真的不想執行「sleep XXX」，想改成「echo hello」，可以這樣下達指令:
```bash
docker run --entrypoint echo my-sleep hello
```

## Command & Arguments in Kubernetes

了解了`Docker`中的`command`與`arguments`後，我們來看看`k8s`中的設定:

  
假如我想要在`Pod`中執行`sleep 3600`，我以這樣編寫我的`yaml`檔:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-sleep
spec:
  containers:
  - name: my-sleep
    image: my-sleep
```

 > 因為使用的`image`是「my-sleep」，也就是上面打包的`image`，當然預設就是執行「sleep 3600」。

那如果今天我想要縮短休眠時間為1分鐘，就必須`args`為60，取代原本的「3600」:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-sleep
spec:
  containers:
  - name: my-sleep
    image: my-sleep
    args: ["60"]
    ```
```

假設今天我想執行「echo hello」，可以這樣寫:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-sleep
spec:
  containers:
  - name: my-sleep
    image: my-sleep
    command: ["echo"]
    args: ["hello"]
```

**補充**

如果要一次執行多條指令，可以這樣寫:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-sleep
spec:
  containers:
  - name: my-sleep
    image: my-sleep
    command: ["/bin/sh", "-c"]
    args: ["echo hello; sleep 300"]
```

從以上的例子，可以看出`Dockerfile`與`yaml`的對應關係如下:

Dockerfile | yaml
--- | ---
ENTRYPOINT | command
CMD | args


> 在`yaml`中，`command`會取代原先`image`中的`ENTRYPOINT`，而`args`會取代原先`image`中的`CMD`。


## 練習1

請創建一個yaml來建立一個`Pod`，需求如下:
  * `name`: busybox
  * `image`: busybox
  * 在容器內執行指令: `echo hello; sleep 300`

## 練習2

只能使用指令建立一個`Pod`，需求如下:
  * `name`: alpine
  * `image`: alpine
  * 在容器內執行指令: `echo hello; sleep 300`