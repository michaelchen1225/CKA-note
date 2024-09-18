### 今日目標

---
* Pod 的除錯

* Networking 的除錯

* Master Node 的除錯

* kube-apiserver 的除錯

* Node 的除錯

* kubelet、container runtime 的除錯

* Event 的觀察

***
  
終於來到了最後一個章節的最後一篇，[昨天](https://ithelp.ithome.com.tw/articles/10352785)我們提到有效的監控才能即時發現問題，今天我們來講「解決問題」，也就是 Troubleshooting。


## Debug Pod

我們將 Pod 建立後，最常使用 `kubectl get pod` 來查看 Pod 是否正常運作。如果發現錯誤時，有兩大方式可以了解問題的原因：

1. **`kubectl describe pod <pod-name>`**

這個指令會列出 Pod 的詳細資訊，需重點觀察的欄位有：

* **Status**：Pod 的狀態，讓你馬上清楚 Pod 是 Running、Pending 等等

* **Containers**：
  * **State**：Container 的狀態，例如 waiting、running、terminated，並且可以透過底下的「`Reason`」了解原因。
  * **Last State**：如果 Container 曾經重啟過，這裡會顯示上次執行的狀態，同樣可以透過「`Reason`」了解原因。
  
* **Conditions**：有以下五大類，以「True」或「False」表示：

  | Type | 描述 |
  | --- | --- |
  | PodReadyToStartContainers | Pod 的 sandbox 與網路設定是否已完成 |
  | PodScheduled | Pod 是否已經被安排至 Node 上 |
  | Initialized | init container 是否已成功執行完畢 |
  | ContainersReady | Pod 終所有 container 是否已準備好處理 requests |
  | Ready | Pod 是否已準備好處理 requests |

* **Events**：可以看到 Pod 的建立過程中發生了什麼事，例如 Pod 被調度到哪個 Node、Image 下載是否成功、Container 啟動是否成功等等。


2. **`kubectl logs <pod-name>`**

這個指令可以查看 Pod 中 container 的 log，也就是 container 的 stdout 和 stderr。如果 Pod 啟動失敗是因為 container 內部的問題，可以用這個指令來了解原因。

常見的 logs 指令如下：

* 抓出 Pod 中 container 的 log：

```bash
kubectl logs <pod-name>
```

* 若想「持續」的將 log 輸出，可以使用加上「`-f`」選項：

```bash
kubectl logs -f <pod-name>
```

* 如果 Pod 中有多個 container，可以使用「`-c`」指定：

```bash
kubectl logs -f <pod-name> -c <container-name>
```

* 也可以一次性列出「所有」 container 的 log：

```bash
kubectl logs -f <pod-name> --all-containers=true
```

* 如果遇到不斷重啟失又敗的 Pod，可以使用「`--previous`」參數來看上一次重啟前的 log：
```bash
kubectl logs --previous <pod-name>
```

* 也可以以時間範圍來篩選 log，例如我想抓出「前一個小時」產生的 log：
```bash
kubectl logs --since=1h <pod-name>
```

* 僅列出最後幾行的 log，例如列出最後 10 行：
```bash
kubectl logs --tail=10 <pod-name>
```

* 查看 deplooyment 第一個 Pod 中的 log：
```bash
kubectl logs deploy/<deployment-name>
```
> 同理，查看 daemonset Pod 的 log 就是 `kubectl logs daemonset/<daemonset-name>`，replicaSet、statefulSet、service 就以此類推。

* 查看 deployment 中的所有 pod 的 log：
```bash
kubectl logs deploy/<deployment-name> --all-containers=true
```

> 如果容器根本就無法啟動，就算用 `kubectl logs` 也看不到 log，因此得回去用 `kubectl describe pod` 來找問題。

但如果遇到比較糟糕的情況，例如 kube-apiserver 掛掉，根本無法使用 kubectl 指令，這時就只能回到 container runtime 的 log 中去查找問題：

* 首先，找出壞掉的 container id：

```bash
docker ps -a
```
或是：

```bash
crictl ps -a
```
> 必須加上「-a」，否則只會顯示正在執行的 container

* 找到 id 後，查看容器的 log：
  
```bash
  docker logs <container-id> 
```
或是:
```bash
  crictl logs <container-id>
```
> \<container-id> 只需要輸入前面幾個字元即可，不一定要把 id 全部輸入


不過如果容器不斷重啟，用上面的方式也看不到 Log，因為被重啟洗掉了。這時可以直接到容器的「log 檔」中去查找原因：

  * `Docker`：/var/lib/docker/containers/\<container-id>/xx.log

  * `containerd`：/var/log/pods/\<container-name>/xx.log

> 注意：Pod 中容器的 log 檔存在於「Pod 被調度到的 Node 上」，因此得先知道 Pod 究竟在哪個 Node 上，在 ssh 過去找上述 log 檔的位置。

---
**這裡統整一下 Pod 的除錯方針：**

* 如果 kubectl 可以使用，先用 `kubectl get pod` 大致了解 Pod 的狀態。

  * 如果看到「Pending」，表示容器還沒啟動，只能用 `kubectl describe pod` 來查看原因。
  * 如果看到「CrashLoopBackOff」，表示容器經歷不斷重啟，可以用 `kubectl logs` 來查看容器的輸出。
  * 如果是 「ImagePullBackOff」等 image 相關的輸出，就直接修正 image 的問題即可，比較直覺。

> 通常除了 Pending 以外的錯誤都能用 `kubectl logs` 來找原因。

* 如果 kubectl 無法使用，就直接到 container runtime 的 log 中去查找問題。
  * 使用 `docker ps -a` 或 `crictl ps -a` 找出壞掉的 container id --> 查看容器的 log
  * 直接到容器的「log 檔」中去查找原因。
***

## Debug Networking

**Service 相關問題**

* 如果存取 Service 失敗，可以先看看 Service 究竟有沒有抓到 endpoints：

```bash
kubectl get ep <service-name>
```

* 如果抓不到 endpoints，通常是 Label Selector 的問題，可以用直接用 edit 來修改：

```bash
kubectl edit svc <service-name>
```

另外，Service 的 logs 其實等同於 endpoint 的 logs，如果只知道 Service 的名稱，可以用以下指令找出 endpoint 的 logs：

```bash
kubectl logs svc/<service-name>
```

**Pod 相關的網路問題**

* 在同一個 Pod 內，只能有「一個」容器 listen 在一個 port 上，例如已經有一個容器 listen 在 80 port 上，另一個容器就不能再 listen 80 port ，否則另一個容器啟動後會進入出現 Error。

* 當兩個 Pod 之間連線失敗時，在確定兩個 Pod 的設定沒有問題後，如果還是無法連線，可以檢查一下 kube-proxy：

```bash
kubectl get daemonset -n kube-system kube-proxy
```
> 如果發現 kube-proxy 出問題了，就用上面 Pod 的除錯方法來查看原因。

* 接續上面的情境，若兩個 Pod 都用 Service 來 expose，可以確認一下 Service 是否有抓到 endpoints：

**CNI 相關問題**

* 通常 CNI 通常是 daemonset，可以用以下指令來查看 CNI 的狀態：

```bash
kubectl get daemonset -n kube-system <CNI-name>
```
> 如果有問題就一樣回到上面 Pod 的除錯方法來查看原因。

* CNI 的重要設定檔位置：
  * 目前系統支援的 CNI 執行檔：`/opt/cni/bin/`
  * CNI 的設定檔：`/etc/cni/net.d/`


## Debug Master Node

Master Node 也稱為 Control Plane，是整個 cluster 的核心，上面包含了幾個重要的元件：kube-apiserver、etcd、kube-scheduler、kube-controller-manager、kubelet。

Cluster 的重要元件都會有固定的 port：

  * **kube-apiserver**：6443 
  * **etcd**：2379-2380
  * **kube-scheduler**：10259
  * **kube-controller-manager**：10257
  * **kubelet**：10250

> etcd 的 2379 是 client port，2380 是 peer port (多台 etcd server 之間通訊的 port)

這些 port 如果忘記的話，通常都可以在上述元件的 static pod yaml 中找到，或者用 netstat 來找：
```bash
netstat -tulnp | grep <api/etcd/scheduler/controller/kubelet>
```

知道這些 port 有什麼用呢？用處可大了，能讓我們在除錯時更快的得到線索，例如今天 kube-apiserver 出錯了，其中一條 log 長這樣：

```text
2024-03-13T12:41:01.395950398Z stderr F W0313 12:41:01.395821       1 logging.go:59] [core] [Channel #198 SubChannel #199] grpc: addrConn.createTransport failed to connect to {Addr: "127.0.0.1:2379", ServerName: "127.0.0.1:2379", }. Err: connection error: desc = "transport: Error while dialing: dial tcp 127.0.0.1:2379: connect: connection refused"
```

可以看到錯誤原因是無法連線到 etcd (127.0.0.1:**2379**)，這時就可以去檢查 etcd 的狀態了。

> 除了連線問題之外，Master Node 上的重要元件的設定會透過 configmap 或 volume 掛載進去，在除錯時可以特別留意一下。

### 關於 kube-apiserver 的除錯

通常 Master Node 上的重要元件出錯，都可以用上面介紹【Debug Pod】的方法來查看 log，這裡我們重點來看看 kube-apiserver 的除錯。

kube-apiserver 一旦壞掉，kubectl 是無法使用的，因此只能在沒有 kubectl 的情況下來找原因，常見的方法如下：

* 直接看 kube-apiserver 容器的 log 檔案：

  * `Docker`：/var/lib/docker/containers/\<container-id>/xx.log

  * `containerd`：/var/log/pods/\<kube-system_kube-apiserver-xxxx>/xx.log

* 不過有些 cluster 中的 kube-apiserver 不是用 static pod 跑的，而是用 systemd 服務跑，則可以這樣找 log：
```bash
journalctl -u kube-apiserver.service
```

有的時候，如果有人更動了 kube-apiserver 的 static pod yaml，造成 yaml 格式出問題的話，我們甚至連容器的 log 都看不到，那該怎麼辦呢？我們知道 static pod 是 kubelet 管的，因此我們可以用 `journalctl` 來看 kubelet 的 log：

```bash
journalctl -u kubelet.service | grep "apiserver"
```

### Debug Worker Node

以下列出幾個 Worker Node 可能遇到的情況：

* 假設明明就有規劃某個 Worker Node 成為 cluster 的一部分，但是執行 `kubectl get nodes` 卻發現該 Node 不在列表中，可能是因為還沒「join」的原因，我們可以在 Master Node 上來產生 join command：

```bash
kubeadm token create --print-join-command
```

* 如果執行 `kubectl get nodes` 發現某個 Node 的狀態是「NotReady」，可以先 ssh 進去該 Node 上，檢查 `
kubelet` 和 `container runtime` 是否正常運作：

```bash
systemctl status kubelet.service
systemctl status containerd.service 
```

* 如果發現 `kubelet` 或 `container runtime` 有在執行，持續的輸出 log 來觀察看看

```bash
journalctl -u kubelet.service -f
journalctl -u containerd.service -f
```

* 如果發現 `kubelet` 或 `container runtime` 是 inactive (dead) 的狀態，先嘗試重啟看看：
```bash
systemctl restart kubelet.service
systemctl restart containerd.service
```

* 如果啟動後仍然發現 `kubelet` 或 `container runtime` 是「dead」的狀態，代表可能是服務設定檔出問題了，可以用 `journalctl` 來看 log：
```
journalctl -u kubelet.service
journalctl -u containerd.service
```

* 如果確定是設定檔的問題，但不清楚設定檔的具體位置，在 `systemctl status` 的輸出中找到：

```bash
systemctl status kubelet.service
```
```text
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
```
> 這裡統整一下 kubelet 的重要設定檔位置：

* **/lib/systemd/system/kubelet.service**：啟動腳本
* **/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf**：kubelet 的額外設置
* **/var/lib/kubelet/config.yaml**：kubelet 的參數設定檔

如果有修改任何設定檔，重啟服務時要記得「daemon-reload」：

```bash
systemctl daemon-reload
systemctl restart kubelet.service
```

## Event 的觀察

`Event` 是一種 K8s 物件，用來記錄整個 cluster 中發生的重要事件，能夠幫助我們了解 cluster 中的狀態。

> 不過 Event 並不是一個「持久」的物件，只會保留一段時間，這點要特別注意。

以下列出幾個關於 Event 的常見操作：

* 列出整個 cluster 的 Event：

```bash
kubectl get events -A
```
輸出：
```test
NAMESPACE   LAST SEEN   TYPE      REASON      OBJECT     MESSAGE
default     5m9s        Normal    Scheduled   pod/test   Successfully assigned default/test to node01
default     4m56s       Normal    Pulling     pod/test   Pulling image "test"
default     4m55s       Warning   Failed      pod/test   Failed to pull image "test"...
...(省略)
```
> 輸出相當的一目瞭然，可以知道這個 event 發生在哪個 namespace、什麼時間、發生的類型、原因、主體、訊息等等。

* 列出某個 namespace 的 Event：

```bash
kubectl get events -n <namespace>
```

* 持續的列出 Event：

```bash
kubectl get events -w
```

* 列出 Event 詳細資訊，重要的是能知道 Event 的是由誰產生的(Source)：
```bash
kubectl get events -o wide
```

> 延伸應用：例如查看 kubelet 產生的 Event：

```bash
kubectl get events -o wide | grep kubelet
```

* 根據時間先後排序 Event：

```bash
kubectl get events --sort-by='{.metadata.creationTimestamp}'
```

### 今日小結

今天是「Troubleshooting & Monitoring」的最後一篇，介紹了一些除錯的小技巧，重點在於「如何找到問題」。其實除錯的最重要的是心平氣和，有時候就只是打錯字而已，通常靜下來慢慢看都能找到問題。

那本次鐵人賽關於 k8s 入門的六個章節就全部結束了，明天會來更新最後一篇文章，內容關於 CKA 考試的攻略、心得等等，如果是正在準備 CKA 的讀者可以參考看看喔。  

-----
**參考資料**

* [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

* [Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)

* [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)


