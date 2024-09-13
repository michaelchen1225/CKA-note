### 今日目標

* drain & cordon 的操作
* Upgrade kubeadm cluster

今天是「Cluster Architecture, Installation & Configuration」的第一篇，關於 cluster architecture & installation 的部分，我們已經在 [Day 02](https://ithelp.ithome.com.tw/articles/10345505) & [Day 03](https://ithelp.ithome.com.tw/articles/10345660) 中介紹過了，今天要來談談 cluster 的**升級**。

k8s 大約每三個月就會有些更新釋出，定期留意官方版本的更新狀態是個不錯的習慣。在 [Day 03](https://ithelp.ithome.com.tw/articles/10345660) 中，我們使用 kubeadm 來建立 cluster ，因此今天就來介紹如何使用 kubeadm 升級 cluster。

升級 cluster 時，正在被升級的 Node 上基本上是無法跑任何 Pod 的，且除了 cluster 升級之外，Node 在升級作業系統或需要維護時也無法讓 Pod 在上面執行，那麼在 Node 因為更新或維護而不可用時，「正在執行的 Pod」會被如何處理呢？這裡得先談談「Pod eviction timeout」的概念。


## Pod eviction timeout

當 Node 因為某些原因而無法使用時，正在該 Node上執行的 Pod 並不會立刻刪除， kubelet 會等待一段時間，如果在這段時間內 Node 恢復正常，Pod 就會繼續執行，否則 kubelet 會將 Pod 刪除或驅逐(evict)。

---
> **Tips：被驅逐還是刪除？** 

首先，得先釐清「驅逐」的定義：*Pod 被移到其他 Node 上繼續執行。*

所以，「驅逐」與「刪除」還是有差別的，那究竟會被驅逐還是被刪除，取決於 Pod 是否屬於某個 ReplicaSet 或 Deployment：

 * 若 Pod 屬於某個 ReplicaSet 或 Deployment，會被驅逐。
 * 若 Pod 不屬於任任何 ReplicaSet 或 Deployment，會被刪除。

 > 如果重要的 Pod 用 ReplicaSet 或 Deployment 執行會比較保險。
***

而 kubelet 「等待 Node 恢復正常的這段時間」，就是 Pod eviction timeout，這個時間預設是 5 分鐘，可以在 kube-controller-manager 的參數中調整:

```yaml
--pod-eviction-timeout=5m0s
```

但在 Pod eviction timeout 的期間，Pod 所提供的服務就無法被使用，因為即使是會被驅逐的 Pod，預設上也是要等到 5 分鐘之後才能在其他 Node 上跑起來。何況天有不測風雲，要是 Node 能回的來那還好說，就怕出了甚麼意外，因此今天要來介紹 k8s 的兩種機制：drain 和 cordon。

### Drain

`drain` 是一個 kubectl 的指令，會將 Node 標記為「Unschedulable」(不會有 Pod 被調度到該 Node 上)，並將本來就在該 Node 上執行的 Pod 「驅逐」至其他 Node：

```bash
kubectl drain <node-name>
```

Drain 完 之後就可以開始對 Node 進行更新、維護。之後等 Node 重新準備好後，可以使用 uncordon 指令將 Node 重新標記為 「Schedulable」：
```bash
kubectl uncordon <node-name>
```

> 不過，被驅逐或刪除的 Pod 在 Node 回來後並不會自動返回，除非後續重新部署並安排到該 Node 上。


最後，使用 drain 時有以下兩點需要注意：

  * 如果有「並非使用 replication controller、replicaSet、daemonset、stateful set、job 建立」的 Pod ，則 drain 會失敗，需要加上「--force」才會**刪除**(而非驅逐)。

```bash
kubectl drain <node-name> --force
```

  * 如果 Node 上有 daemonset 則 drain 會失敗，需要加上「--ignore-daemonsets」，讓這些 daemonset 繼續執行：

```bash
kubectl drain <node-name> --ignore-daemonsets
```

> 如果已經決定要徹底 drain 某個 Node，可以直接下達：
```bash
kubectl drain <node-name> --force --ignore-daemonsets
```


### 測試 Drain

> 以下測試環境為 killercoda，所以 Master Node 沒有 taint，Pod 可以被部署在上面。

* 先部署一個 Pod：
```bash
kubectl run nginx --image=nginx
```

* 再部署一個 Deployment：
```bash
kubectl create deployment web --image=nginx --replicas=2
```

* 查看 Pod 執行在哪個 Node 上：
```bash
kubectl get pod -o wide
```
```text
NAME                   READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
nginx                  1/1     Running   0          2m18s   192.168.1.5   node01         <none>           <none>
web-7c56dcdb9b-2nqkz   1/1     Running   0          23s     192.168.1.6   node01         <none>           <none>
web-7c56dcdb9b-7ghhn   1/1     Running   0          23s     192.168.0.5   controlplane   <none>           <none>
```

* 可以看到部分 Pod 跑在 node01 上，我們 drain 掉 node01 試試看：
```bash
kubectl drain node01 --force --ignore-daemonsets
```

* 查看 Pod 被刪除或驅逐的情況：
```bash
kubectl get pod -o wide
```
```text
NAME                   READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
web-7c56dcdb9b-7ghhn   1/1     Running   0          3m54s   192.168.0.5   controlplane   <none>           <none>
web-7c56dcdb9b-8g5kc   1/1     Running   0          19s     192.168.0.6   controlplane   <none>           <none>
```

因為 「web」屬於 Deployment，所以被驅逐到其他 Node 上，而不屬於 Deployment 也不屬於 ReplicaSet 的 「nginx」則被刪除。

* 最後 uncordon node01，讓它變回「Schedulable」：
```bash
kubectl uncordon node01
```

### Corden

Cordon 則是將 Node 標記為「SchedulingDisabled」，也就是「新的 Pod」不會被安排到該 Node 上，但 cordon 不會驅逐或刪除舊的 Pod，且舊的 Pod 仍會繼續執行：
```bash
kubectl cordon <node-name>
```

當 Node 的維護完畢後，同樣恢復成 Schedulable：
```bash
kubectl uncordon <node-name>
```

### 測試 Cordon

* 部署一個 Pod：
```bash
kubectl run busybox --image=busybox --command -- sleep 3600
```

* 再部署一個 Deployment：
```bash
kubectl create deployment frontend --image=nginx --replicas=2
```

* cordon node01：
```bash
kubectl cordon node01
```

* 查看 Pod 的情況：
```bash
kubectl get pod -o wide
```
```text
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
busybox                     1/1     Running   0          84s   192.168.1.4   node01         <none>           <none>
frontend-56dc7ffc86-d2hpn   1/1     Running   0          77s   192.168.0.4   controlplane   <none>           <none>
frontend-56dc7ffc86-xlxwj   1/1     Running   0          77s   192.168.1.5   node01         <none>           <none>
```

可以看到 Pod 仍然在 node01 上，但是 node01 已經被標記為 「SchedulingDisabled」：

```bash
kubectl get node -o wide
```
```text
NAME     STATUS                     ROLES    AGE    VERSION
node01   Ready,SchedulingDisabled   <none>   6d1h   v1.30.0
```

了解 drain 和 cordon 的概念後，接下來就可以開始升級 cluster 了。

## Cluster Upgrade strategy

粗略的說，升級 cluster 有這兩大階段：

  **STEP 1.** 升級 Master Node (controlplane)
   
  **STEP 2.** 升級 Worker Node

至於 worker node 的更新，有以下幾種策略可以選擇：

  1. 一次性關閉所有 Worker Node，然後一次更新完畢。

  2. 一次 drain 一個 Worker Node，依序更新直到全部更新完畢。

  3. 準備一個新的 Worker Node，將 Pod 移到新的 Node 之後，淘汰一個舊的 Worker Node，如此循環直到全部更新完畢。

在雲端的環境中，第三種策略是相當常見的，因為新增一台新的機器只需要動動手指就搞定了。那如果是非雲端的環境，第一種方式雖然簡單，但是會造成服務的中斷，因此第二種方式比較常見，底下以第二種方式來示範。

**升級目標**：將 cluster 從 v1.30.0 升級到 v1.31.0

> 如果沒有 v1.30.0 的環境，可以到 killercoda 上開一個環境來測試。另外，如果你想升級不是 v1.30.0 --> v1.31.0 也沒關係，因為升級過程大同小異，注意版本號即可。(目前文章時間為 2024 年九月)

### STEP 1. 升級 Master Node**

> 在 Master Node 上進行以下操作：

* 確認目前 cluster 的版本:

```bash
kubectl get node
```
```text
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   2d15h   v1.30.0
node01         Ready    <none>          2d15h   v1.30.0
```

* Drain Master Node：
```bash
kubectl drain controlplane --ignore-daemonsets
```

* 確認目前 kubeadm 的 apt repository 版本：

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```
```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
```

* 把 v1.30 改成 v1.31：

```bash
sudo sed -i 's/v1.30/v1.31/g' /etc/apt/sources.list.d/kubernetes.list
```
---
> **Tips**

如果你使用的是 killercoda，你可能會看到 /etc/apt/sources.list.d/kubernetes.list 的內容是這樣:

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-1-28-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-1-29-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-1-30-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
```

這時請這樣修改：
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

改完後確認一下:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```
```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /
```
***


* 改完  apt repository 之後，確認可用的新版本：
```bash
sudo apt update
sudo apt-cache madison kubeadm
```
```text
   kubeadm | 1.31.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
   kubeadm | 1.31.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.31/deb  Packages
```

* 這裡選擇用 `1.31.1-1.1` 版本來升級。先升級 kubeadm：
```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.31.1-1.1*' && \
sudo apt-mark hold kubeadm
```

* 更新後確認 kubeadm 的版本：

```bash
kubeadm version
```
```
kubeadm version: &version.Info{Major:"1", Minor:"31", GitVersion:"v1.31.1", GitCommit:"948afe5ca072329a73c8e79ed5938717a5cb3d21", GitTreeState:"clean", BuildDate:"2024-09-11T21:26:49Z", GoVersion:"go1.22.6", Compiler:"gc", Platform:"linux/amd64"}
```

* 可以看到 kubeadm 的版本已經更新到 `v1.31.1`。接著確認升級的可行性：

```bash
kubeadm upgrade plan
```

* 會看到類似以下的畫面：
```text
......
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE           CURRENT   TARGET
kubelet     controlplane   v1.30.0   v1.31.0
kubelet     node01         v1.30.0   v1.31.0

Upgrade to the latest stable version:

COMPONENT                 NODE           CURRENT    TARGET
kube-apiserver            controlplane   v1.30.0    v1.31.0
kube-controller-manager   controlplane   v1.30.0    v1.31.0
kube-scheduler            controlplane   v1.30.0    v1.31.0
kube-proxy                               1.30.0     v1.31.0
CoreDNS                                  v1.11.1    v1.11.3
etcd                      controlplane   3.5.12-0   3.5.15-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.31.0

Note: Before you can perform this upgrade, you have to update kubeadm to v1.31.0.

_____________________________________________________________________

```


* 出現以上畫面就可以依照提示升級 Master Node 了：

```bash
sudo kubeadm upgrade apply v1.31.1
```
```text
...
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.31.1". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
> 等待五分鐘左右，看到以上提示就代表 master node 已經升級完畢 (包括重要元件)

* 接著更新 kubelet 與 kubectl：
```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.31.1-1.1*' kubectl='1.31.1-1.1*' && \
sudo apt-mark hold kubelet kubectl
```

* 然後重啟 kubelet：
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

這樣 Master Node 就升級完畢了。確認一下：

```bash
kubectl get node
``` 
```text
NAME           STATUS                     ROLES           AGE     VERSION
controlplane   Ready,SchedulingDisabled   control-plane   2d15h   v1.31.1
node01         Ready                      <none>          2d15h   v1.30.0
```

接著讓 Master Node 恢復成 Schedulable：
```bash
kubectl uncordon controlplane
```

準備更新 Worker Node，先 drain node01：
```bash
kubectl drain node01 --ignore-daemonsets
```

最後 ssh 到 Worker Node 上：
```bash
ssh node01
```

### STEP 2. 升級 Worker node

> 在 Worker Node** 上進行以下操作

* 同樣查看一下 apt repository 的版本:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```
```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
```

* 把 v1.29 改成 v1.30：

```bash
sudo sed -i 's/v1.30/v1.31/g' /etc/apt/sources.list.d/kubernetes.list
```
---
> Tips

killercoda 的修改方式：
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

* 改完 apt repo 後確認一下：

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```
```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
```
***

* 先更新 kubeadm：
```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.31.1-1.1*' && \
sudo apt-mark hold kubeadm
```

* 升級 worker node：

```bash
sudo kubeadm upgrade node
```
輸出：

```text
......(省略)
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```
> 輸出的最後兩行說明 Worker Node 已經升級完畢。更新時間上 Worker Node 會比 Master Node 快很多，因為不用更新 api-server、controller-manager 等等重要元件。 

* 更新 kubelet 和 kubectl (其實步驟和 master node 很像)：
```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.31.1-1.1*' kubectl='1.31.1-1.1*' && \
sudo apt-mark hold kubelet kubectl
```

* 重啟 kubelet：
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

這樣整個 cluster 就升級完畢了。exit 回到 Master Node 確認一下：
```bash
kubectl get node
```
```text
NAME           STATUS                     ROLES           AGE     VERSION
controlplane   Ready                      control-plane   2d15h   v1.31.1
node01         Ready,SchedulingDisabled   <none>          2d15h   v1.31.1
```

最後將 node01 恢復成 Schedulable：
```bash
kubectl uncordon node01
```

### 今日小結

今天示範了如何升級一個使用 kubeadm 建立的 cluster，如果是正在準備 CKA 的讀者，今天的內容基本必考，忘記更新步驟其實沒關係，反正考試可以看[官網](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)，重點是熟悉整個升級的流程。


----

**參考資料**

* [kubectl drain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/)

* [kubectl cordon](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_cordon/)

* [drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain)

* [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
