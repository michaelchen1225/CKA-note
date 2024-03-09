# Drain & Cordon

當`node`上的作業系統需要升級或需要維護時，會暫時關閉一段時間，那麼正在運行的`pod`會被如何處理呢? 這裡我們得先談談`pod eviction timeout`的概念。

## Pod eviction timeout

當`node`因為某些原因而無法使用時，上面的`pod`並不會立刻刪除，`kubelet`會等待一段時間，如果在這段時間內`node`恢復正常，`pod`就會繼續運行，否則`kubelet`會將`pod`刪除或在驅逐(evict)至其他`node`上重新建立。

> 是驅逐還是刪除，取決於`pod`是否歸屬於某個`ReplicaSet`或`Deployment`，若「是」則被驅逐，否則會被刪除

而`kubelet`等待的這段時間，就是`pod eviction timeout`，這個時間預設是`5分鐘`，可以在`kube-controller-manager`的參數中調整:
```yaml
--pod-eviction-timeout=5m0s
```

但就算有`eviction timeout`，在這段時間內`pod`所提供的服務還是無法被使用，更何況天有不測風雲，要是`node`能回的來那還好說，就怕出了甚麼意外，因此今天要來介紹`k8s`的兩種機制: `drain`和`cordon`。

## Drain & Cordon

`drain`是一個`kubelet`的指令，會將`node`標記為`Unschedulable`，並將上面的`pod`驅逐至其他`node`:
```bash
kubectl drain <node-name>
```

不過要注意以下幾點:
  * 如果有「不是使用`replication controller`、`replica set`、`daemon set`、`stateful set`、`job`建立」的`pod`，則`drain`會失敗，需要加上`--force`參數才會**刪除**:

```bash
kubectl drain <node-name> --force
```

  * 如果`node`上有`daemon set`，則`drain`會失敗，需要加上`--ignore-daemonsets`參數:

```bash
kubectl drain <node-name> --ignore-daemonsets
```

至於`cordon`，則是將`node`標記為`Unschedulable`，但不會驅逐或刪除`pod`:
```bash
kubectl cordon <node-name>
```

當該做的維護完畢後，就可以將被`drain`或`cordon`的`node`恢復成`Schedulable`:
```bash
kubectl uncordon <node-name>
```

> 被驅逐的`pod`並不會自己回來，只有重新部署才有可能會到原本的`node`

### 練習1

以下練習將會用到兩個`node`:
  1. controlplane
  2. node01

建立一個`deployment`，需求如下:
  * `name`: nginx
  * `image`: nginx
  * `replicas`: 3

再建立一個`pod`，需求如下:
  * `name`: httpd
  * `image`: httpd
  
建立兩者後，移除`contrlplane`上的`taint`，並將`node01`標記為`Unschedulable`，最後觀察`pod`的狀態


### 練習2

接續上一題，將`node01`標記為`Unschedulable`並刪除或驅逐`pod`，並觀察`pod`的狀態


[參考解答](18-2.md)