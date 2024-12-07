### 今日目標

1. Kubernetes 的基本執行單位 --- Pod

2. Kubernetes 的架構 --- Cluster

3. Cluster 的基本硬體單位 --- Node

  * Node 的必要組件：kubelet、container runtime、kube-proxy

  * Node 的角色：Master Node、Worker Node

  * Master Node 的特殊組件：kube-apiserver、etcd、kube-scheduler、kube-controller-manager

3. 了解 HA cluster 的設計方式。


> 底下會嘗試用一個**船隊**的比喻，來解釋 Kubernetes 的架構與組件。

那就讓我們從 Kubernetes 最基本的**執行單位**談起吧。

## 基本的執行單位 --- Pod

K8s 是一個容器管理平台，這些被管理的容器(container)會在 **Pod** 中執行：

   * 一個 Pod 能跑 1 個或多個容器，視情況選擇：
     * 功能僅需一個容器：Single-container Pod
     * 功能需要多個容器協同合作：Multi-container Pod

   * Pod 被產生時會賦予一個「虛擬 IP」，用來與其他 Pod 溝通。

下圖為 Pod 與 container 的關係：

![https://ithelp.ithome.com.tw/upload/images/20240820/201686927R7EfN2nGm.png](https://ithelp.ithome.com.tw/upload/images/20240820/201686927R7EfN2nGm.png)

> 以船隊的比喻來說， Pod 就是船上的貨櫃，而容器就是貨櫃裡面的貨物。

那麼這些 Pod 又是如何被 K8s 管理呢？在回答這個問題之前，我們得先了解 K8s 的基本架構，也就是 cluster。

## Kubernetes 的架構 --- Cluster

「**Cluster** (叢集)」是一種 topology。當你部署了 Kubernetes，一個由「**Node** (節點)」組成的 cluster 就形成了。

對 K8s cluster 來說，這些 Node 的功能如下：

   * 一個 Node 代表一台伺服器(無論是實體還是虛擬的)，為負責提供執行環境給 Pod。

> 如果 cluster 想像成一個船隊，「船」就是 Node，船上載著貨櫃 (Pod)

## Cluster 的基本硬體單位 --- Node

每個 Node 的必要組件如下：

1. **Kubelet** 

   相當於每艘船 (也就是 Node) 上的**船長**，負責執行總指揮 (Master Node，等等會提到) 傳遞的命令。例如執行、刪除 Pod 等等。

2. **Container Runtime**

   在 Pod 中執行的容器，都需要容器的「執行引擎」才能跑起來，這就是「container runtime」，常見的有Containerd、CRI-O 等。

> 常聽到的 Docker 並不是 Container Runtime，而是「包含」著 Container Runtime 的容器管理工具，兩者間的關係可以參考[這篇文章](https://bluelight.co/blog/containerd-vs-docker#containerd-vs-docker-a-head-to-head-comparison)

3. **Kube-porxy**

   Cluster 內部的 Pod 可以透過內部網路互相溝通，而 kube-proxy 則負責維護 cluster 內部網路的「路由規則」。

而因為 Node 的角色不同，又可分為以下兩種：

* **Master Node** 

   又稱為「**Control Plane**」，負責管理與指揮整個 cluster，例如資源調度、cluster 的狀態監控等等。
   
   管理員可以透過 CLI (Command Line Interface) 或 API 來下達指令或任務給 Master Node。

* **Worker Node** 

   Pod 會被 Master Node 調度到 Worker Node 上執行，所以 Worker Node 需負責提供 Pod 的執行環境，並在 Master Node 的指揮下執行任務。

> 如果 cluster 想像成一個船隊，那麼其中負責管理、發號施令的主船，就稱為 Master Node，負責接收主船命令並執行的小船，就稱為 Worker Node。

### Master Node 的特殊組件

Master Node 身為整個船隊的**總指揮**，除了擁有上述提到的三個組件(kubelet、container runtime、kube-proxy)外，還有**額外四個**特殊的組件：

1. **kube-apiserver**

    在整個 cluster 中，任何訊息的傳遞，都必須經過 kube-apiserver 的轉介。例如管理者對整個 cluster 下的指令、Node 與 Node 之間的溝通等等。訊息傳遞者的身分與授權，都屬於 kube-apiserver 的管轄範圍。
    
    萬一 kube-apiserver 發生故障，就等於 cluster 的通訊系統癱瘓了，所以 kube-apisever 是一個極為重要的組件。

2. **etcd** 

   以 key-value 的方式存放 cluster 中的資料，例如 cluster 的狀態、目前存在的資源等等。
   
   備份 etcd 是一項重要的工作，因為當整個 cluster 壞掉時，我們可以藉由還原 etcd 的備分來重建 cluster。

3. **kube-scheduler**

   負責 cluster 中資源的調配，例如哪些 Pod 該放到哪個 Node 上。

4. **kube-controller-manager**:

   Cluster 中的資源管理者，是許多控制器(例如: Node-Controller、Replication-Controller)的集合體，透過 kube-apiserver 監控各種資源，並將資源目前的狀態調整至「期望狀態(Desired status)」 。例如 controller-manager 透過 kube-apiserver 發現某個 Pod 壞掉時，controller-manager 會負責重新啟動該 Pod，直到它順利執行。

以上就是關於 Node 以及其組件的大致介紹。如果還是覺得有些混亂的話，這裡我們再次用船隊的比喻總結一下：

> 如果想像整個 cluster 是一個船隊，那麼 Node 就是船、Pod 就是船上的貨櫃，容器就是貨櫃裡的貨物。

> 所有的船 (Node) 都有這三個組件：**kubelet**、**kube-proxy**、**Container Runtime**。

> 在船隊中擔任總指揮的主船 (Master Node)，則另外擁有這四個元件 : **kube-apiserver**、**etcd**、**kube-scheduler**、**kube-controller-manager**。

這裡提供一張圖示：

![https://ithelp.ithome.com.tw/upload/images/20240820/20168692oKw0I7cTyY.png](https://ithelp.ithome.com.tw/upload/images/20240820/20168692oKw0I7cTyY.png)

### HA Cluster

當 cluster 中僅存在一台 Master Node 時，如果 Master Node 發生故障，雖然在 Worker Node 上的 Pod 仍能正常運作，但是 cluster 的管理功能將會癱瘓，例如：因為缺少在 Master Node 上的 kube-controller-manager，所以當 Pod 發生故障時舊無法重啟。

為了避免這種情況，通常在生產環境中會建立一個**高可用性 (High Availability)** 的 cluster，也就是 HA cluster。

HA cluster 擁有**多個** Master Node，避免單點故障的情況發生。除此之外，因為多個 Master Node 的原因，etcd 儲存的資料也會有多個副本，這樣一來即使某個 etcd 發生故障，也能透過其他 etcd 的資料來恢復 cluster。

> 不過，多個 Master Node 上就代表著有多個 api-server，因此 HA cluster 會有一個**負載平衡器 (Load Balancer)** 來分配流量。

在設計 HA cluster 時，依據需求的不同，我們有兩種 topology 可以選擇：

* **Stacked etcd topology**

   Stacked etcd topology **至少**需要三台 Master Node 組成。其實我們前面介紹的就是 stacked etcd，也就是 Master Node 與 etcd 部署在同一台機器上，這樣的好處是容易部署與管理，但必須承擔同時失去 Master Node 與 etcd 的風險。

![https://ithelp.ithome.com.tw/upload/images/20240821/201686925hY76eiygC.png](https://ithelp.ithome.com.tw/upload/images/20240821/201686925hY76eiygC.png)

[圖片來源](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

* **External etcd topology**

  External etcd topology **至少**需要三台 Master Node 與三台 etcd server。 將 etcd 獨立部署在另一台機器上，這樣一這樣的好處是可以避免同時失去 Master Node 與 etcd，但部署複雜度較高，且需額外負擔 etcd server 的成本，比 Stacked etcd 所需的 server 數量多出一倍。
  
![https://ithelp.ithome.com.tw/upload/images/20240821/20168692PsNzn2GdZ3.png](https://ithelp.ithome.com.tw/upload/images/20240821/20168692PsNzn2GdZ3.png)

[圖片來源](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

> 總之，成本與風險之間的考量決定了我們選擇哪一種 topology。

你可能會好奇，為什麼兩種 topology都至少需要**三**個 etcd，而不是兩個、四個或其他數字？這是因為 K8s 採取了 **RAFT** 演算法來保證 etcd 的高可用性與資料一致性，該演算法會從 N 個 etcd 中選出一個作為 Leader，選舉過程大致如下：

1. 在沒有 Leader 的情況下，每個 etcd 都是 Candidate(候選人)。

2. 每個候選人都會設有**隨機**的倒數計時器，等待時間到後，候選人會向其他人發送投票請求。

3. 他人收到投票請求後，若當下還沒有產生 Leader，則投票給該候選人。

4. 若某候選人收到超過半數的選票，則成為 Leader，其餘的則成為 Follower。

5. 新 Leader 會定期發送「心跳訊息」給所有 Follower，用來重置倒數計時器，以免 Follower 重新變成 Candidate。

6. 若 Leader 故障，Follower 沒收到「心跳訊息」會認為沒有 Leader，則回到步驟一重新選舉。

> 可以看出，誰的倒數計時器最快結束，獲選為 Leader 的機率較高。

決定好 Leader 與 Follower 後，當使用者送出「寫入資料的指令」時，會經歷以下步驟：

1. Leader 將該指令寫入自己的 Log 中。

2. Leader 將指令複製並傳送給所有 Follower。

3. Follower 收到指令後，將指令寫入 log 中，並向 Leader 回覆確認。

4. 當 Leader 收到「大多數」，也就是「**(N/2) + 1**」個 Follower 的確認後，Leader 才會執行該寫入指令，並向使用者回覆確認。反之，若 Leader 收到的確認數量小於「**(N/2) + 1**」，則該指令失效，無法進行資料同步！

在這樣的情況下，若 etcd 的個數為偶數，就算只損失一個 etcd，在寫入資料時就不可能滿足「步驟四」的條件，所以 etcd 的個數必須為大於 1 的奇數，通常會以 3 ~ 5 個 Master 來設計 HA cluster。

> 簡單來說，為了避免單一 Master 的風險，我們在權衡風險與成本後，選擇 Stacked etcd 或 External etcd 來設計 HA cluster。又因為 RAFT 演算法，HA cluster 會有奇數個 Master Node，實務上以 3 ~ 5 個為佳。

### 今日小節

今天是 K8s 基礎概念的第一篇，用船隊的比喻，介紹了 K8s cluster 中的基本組件，以及這些組件的功能究竟為何，也另外介紹了 HA cluster 的設計原理，來避免單一 Master 的風險。

了解了 K8s cluster 的架構與組件後，明天我們就會來實際建立一個 cluster ~

---
**參考資料**

* [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

* [Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

* [High Availability Host Numbers](https://discuss.kubernetes.io/t/high-availability-host-numbers/13143)

* [一文彻底搞懂Raft算法，看这篇就够了！！！](https://juejin.cn/post/7218915344130359351)