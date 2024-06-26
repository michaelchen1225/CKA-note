# logging and Monitoring

## logging

當系統中有服務發生問題時，能夠有效的找出原因是相當重要的，而問題的原因往往能在`log`中被找到。我們可以用使用`kubectl logs`指令來查看`pod`的`log`，進而找出問題的原因:

```bash
kubectl logs <pod-name>
```
上述指令只能查看一次性的`log`，如果要持續的監控`log`，可以使用加上`-f`參數:

```bash
kubectl logs -f <pod-name>
```

如果`pod`中有多個`container`，可以使用`-c`參數指定:
```bash
kubectl logs -f <pod-name> -c <container-name>
```

如果遇到不斷重啟失又敗的`pod`，可以使用`--previous`參數來看上一次的`log`:
```bash
kubectl logs --previous <pod-name>
```

但有的時候，`pod`本身因建置失敗而根本無法建立，就無法使用`kubectl logs`來查看了。這時就得到容器的`log`檔案中去查找:

  * 首先，找出壞掉的`container id`:

```bash
# 使用docker為container runtime
docker ps -a
```
or
```bash
# 使用containerd為container runtime
crictl ps -a
```
> 必須加上`-a`參數，否則只會顯示正在運行的`container`

  * 接著，進入容器的`log`:
  
```bash
  docker logs <container-id> 
```
  or
```bash
  crictl logs <container-id>
```
> <container-id> 只需要輸入前面幾個字元即可，不一定要全部打完

另外，也可以直接到容器的`log`檔案中去查找:
  * docker: `/var/lib/docker/containers/<container-id>/`xx.log

  * containerd: `/var/log/pods/<container-name>/`xx.log

**補充**

`k8s`中`master node`元件的`port`對應如下:
  * `kube-apiserver`: 6443 
  * `etcd`: 2379-2380
  * `kube-scheduler`: 10259
  * `kube-controller-manager`: 10257
  * `kubelet`: 10250

稍微記一下上面的port，在查看log的時候會更快地找到問題點，例如今天`kube-apiserver`其中一條log長這樣:
```text
2024-03-13T12:41:01.395950398Z stderr F W0313 12:41:01.395821       1 logging.go:59] [core] [Channel #198 SubChannel #199] grpc: addrConn.createTransport failed to connect to {Addr: "127.0.0.1:2379", ServerName: "127.0.0.1:2379", }. Err: connection error: desc = "transport: Error while dialing: dial tcp 127.0.0.1:2379: connect: connection refused"
```

可以看到錯誤原因是無法連線到`etcd`(127.0.0.1:**2379**)，這時就可以去檢查`etcd`的狀態了。


## Monitoring

發生問題後查找`log`是一種辦法，那「該如何**預防**問題發生」呢? 這取決於管理者對於整體`cluster`資訊的掌握，這時`monitoring`就成為了關鍵。

在`k8s`中，基本的`monitoring`可以用「資源使用情況」來做簡單的衡量，例如:
  * `node`的數量
  * `pod`的數量
  * `CPU`、`Memory的`使用情況

目前市面上有許多第三方的監控工具，例如`Prometheus`、`Grafana`、`metrics-server`等，這些工具能夠幫助管理者監控`cluster`的狀態，而`Kubernetes`本身也有`kube-state-metrics`、`cAdvisor`等工具能夠幫助管理者監控`cluster`的狀態。底下我們將以`matirc server`為例。

### metrics-server的運作原理

在[Day 02](02.md)中，提到每個`node`上都有一個`kubelet`，扮演「船長」的角色。

而`kubelet`底下還有一個子元件: `cAdvisor`。`cAdvisor`是一個`container`的監控工具，監控著`container`的資源使用情況，例如`CPU`、`Memory`、網路等。

當我們在`node`上部署`metrics-server`後，`cAdvisor`會以`summary API`的形式將蒐集到的資訊傳送給`metrics-server`，`metrics-server`再將資訊儲存在**memory**中供使用者透過`API`來查詢，常用的查詢方式如下:

```bash
# 查詢`node`的資源使用情況
kubectl top node

# 指定要查詢的`node`
kubectl top node <node-name>

# 查詢`pod`的資源使用情況
kubectl top pod

# 指定要查詢的`pod`
kubectl top pod <pod-name>
```

> 由於`metrics-server`是將資訊儲存在**memory**中，所以重啟後資料就會消失。不過這可以透過外部儲存來解決。

metrics-server運作原理圖示如下:

![metrics-server](17-1-metric-server.png)

### 部署metrics-server

安裝`metrics-server`非常簡單，我們能從官方的[github](https://github.com/kubernetes-sigs/metrics-server)上找到`yaml`檔案，只要`apply`即可:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

接著就可以使用上面介紹過的`kubectl top`指令來查詢`cluster`的資源使用情況。

# Ref
https://ithelp.ithome.com.tw/articles/10299868
https://www.cnblogs.com/zhangmingcheng/p/15770672.html
https://github.com/kubernetes-sigs/metrics-server
https://ithelp.ithome.com.tw/articles/10247772
https://cloud.tencent.com/document/product/457/35747
