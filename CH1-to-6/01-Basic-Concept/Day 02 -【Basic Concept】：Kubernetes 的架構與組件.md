## 【Basic Concept】：Kubernetes 的架構與組件

## 目錄

* [基本的執行單位 --- Pod](#基本的執行單位-----Pod)

* [Kubernetes 的架構 --- Cluster](#kubernetes-的架構-----cluster)

* [Cluster 的基本硬體單位 --- Node](#cluster-的基本硬體單位-----node)

  * [Master Node 的特殊組件](#master-node-的特殊組件)

* [小結 --- Kubernetes 的架構與組件](#小結-----kubernetes-的架構與組件)

* [HA Cluster](#HA-Cluster)

* [今日小節](#今日小節)

****

在正式開始建置、操作 Kubernetes 之前，我們得先了解 k8s 的整體架構與組件，就像看一本推理小說，得先了解「有哪些角色？」、「這些角色之間的關係？」，才能看懂偵探的推理過程、或自己嘗試推理看看。

> 底下會嘗試用一個**船隊**的比喻，來解釋 Kubernetes 的架構與組件。

那就讓我們從 Kubernetes 最基本的**執行單位**談起吧。

## 基本的執行單位 --- Pod

K8s 是一個容器管理平台，這些被管理的容器(container)會在 **Pod** 中執行：

   * 一個 Pod 能跑 1 個或多個容器，視情況選擇：
     * 功能僅需一個容器：Single-container Pod
     * 功能需要多個容器協同合作：Multi-container Pod

   * Pod 被產生時會賦予一個「虛擬 IP」，用來與其他 Pod 溝通。

> 虛擬 IP 的分配由 CNI (Container Network Interface) 負責，關於這個部分的介紹可以參考 [Day 20](https://ithelp.ithome.com.tw/articles/10348418)。

下圖為 Pod 與 container 的關係：

![https://ithelp.ithome.com.tw/upload/images/20240820/201686927R7EfN2nGm.png](https://ithelp.ithome.com.tw/upload/images/20240820/201686927R7EfN2nGm.png)

> 以船隊的比喻來說，Pod 就是船上的貨櫃，而容器就是貨櫃裡面的貨物。

那麼這些 Pod 又是如何被 K8s 管理呢？在回答這個問題之前，我們得先了解 K8s 的基本架構，也就是 cluster。

## Kubernetes 的架構 --- Cluster

「**Cluster** (叢集)」是一種 topology。當你部署了 Kubernetes，一個由「**Node** (節點)」組成的 cluster 就形成了。

對 K8s cluster 來說，這些 Node 的功能如下：

   * 一個 Node 代表一台伺服器(無論是實體還是虛擬的)，為負責提供執行環境給 Pod。

> 如果 cluster 想像成一個船隊，「船」就是 Node，船上載著貨櫃 (Pod)


## Cluster 的基本硬體單位 --- Node

Node 提供了執行環境給 Pod，而一個 Node 的必要組件如下：

1. **Kubelet** 

   相當於每艘船 (也就是 Node) 上的**船長**，負責執行總指揮 (Master Node，等等會提到) 傳遞的命令。例如執行、刪除 Pod、監控 Pod 的狀態等等。

2. **Container Runtime**

   Kubernetes 透過 Container Runtime Interface 與 Container Runtime 溝通，讓 Container Runtime 執行 Pod 中的容器。這裡的歷史故事其實還蠻有趣的：

   * 2013 年，Docker 出現帶起了一陣容器化的風潮，而也帶來了容器管理的需求，因此 Kubernetes 1.0 也在 2015 應運而生。

   * 作為一個「編排系統」，底層「執行容器」的工作並非由 K8s 自己完成，而是依賴 Container Runtime 處理，而當時最火紅的 Docker 就是不二人選。

     > Docker 本身**並非** Container Runtime，而是「包含」 Container Runtime 的容器管理工具。

   * 但隨著容器技術的發展與多元化，K8s 識到不能被單一的 Docker 綁住，因此於 2016 年推出了自己的 Container Runtime Interface (CRI) 標準，做為與底層 Container Runtime 互動的橋梁。
   
     > 只要你的 Container Runtime 符合 CRI 標準，就能被 K8s 調用。

   * 尷尬的是，由於 Docker 比較早出生，因此並不符合 CRI 標準，因此 K8s 只能先推出一個叫做「Docker shim」的墊片，充當「翻譯員」，讓 K8s 還是能用 CRI 的規範與 Docker 合作：
    
     ```plaintext
     k8s -> <CRI> -> Docker shim -> Docker -> Container Runtime
     ```

     > Docker 本身一堆功能，但其實 K8s 只需要 Docker 的 Container Runtime 而已，再加上 Docker shim 這個「翻譯員」，資源的消耗就變得更多了。

   * 2017 年，Docker 將自家核心的 Container Runtime --- **containerd** 捐給了 CNCF。同為 CNCF 專案的 K8s ，與 containerd 的關係就像是「同門師兄弟」。

   * containerd 身為「師弟」，當然得遵從「師兄」的 CRI 標準，因此 Docker shim 與 Docker 被棄用，改為了 CRI-containerd 與 containerd 的組合：
   
     ```plaintext
     k8s -> <CRI> -> CRI-containerd -> containerd -> Container Runtime
     ```

   * 雖然成功擺脫了 Docker，但還是有 CRI-containerd 這個 daemon 要跑，資源的消耗還有優化空間。因此後來將 CRI-containerd 的功能直接做成 plugin 整合進 containerd 裡，自此 K8s 就可以直接與 containerd 溝通了：

     ```plaintext
     K8s -> <CRI> -> containerd
     ```
   
   * 到了這一步，終於實現了當初的目標：K8s 不被 Docker 綁死，只要你的 Container Runtime 符合 CRI 標準，就能被 K8s 調用。
   
     > 目前最主流的 Container Runtime 有 **containerd**、**CRI-O**等等。

3. **Kube-porxy**

   我們知道容器的生命週期短，所以 IP 經常會因重啟而改變。為了讓大家能用一個穩定的 IP 存取到 Pod，這個穩定的 IP 由 K8s 中的「[Service](https://ithelp.ithome.com.tw/articles/10346530)」提供。
   
   Service 與 Pod 之間的流量轉發，都由 kube-proxy 負責。
   
   kube-porxy 會監控 Service 與 Pod 的變化，並將 Service 的流量正確的轉發到 Pod。

以上為每個 Node 都必須安裝的組件。而因為 Node 的角色不同，又可分為以下兩種：

* **Master Node** 

   又稱為「**Control Plane**」，負責管理與指揮整個 cluster，例如資源調度、cluster 的狀態監控等等。
   
   管理員可以透過 CLI (Command Line Interface) 或 API 來下達指令或任務給 Master Node。
   
   > K8s 的 CLI 是 **kubectl**，[明天](https://ithelp.ithome.com.tw/articles/10345660)會實際安裝。

* **Worker Node** 

   Pod 會被 Master Node 調度到 Worker Node 上執行，所以 Worker Node 需負責提供 Pod 的執行環境，並在 Master Node 的指揮下執行任務。

> 如果 cluster 想像成一個船隊，那麼其中負責管理、發號施令的主船，就稱為 Master Node，負責接收主船命令並執行的小船，就稱為 Worker Node。

### Master Node 的特殊組件

Master Node 身為整個船隊的**總指揮**，除了擁有上述提到的三個組件(kubelet、container runtime、kube-proxy)外，還有額外四個特殊的組件：

1. **kube-apiserver**

    在整個 cluster 中，**任何**訊息的傳遞，都必須經過 kube-apiserver 的轉介。例如管理者對整個 cluster 下的指令、Node 與 Node 之間的溝通、訊息傳遞者的身分驗證與授權，都屬於 kube-apiserver 的管轄範圍。
    
    萬一 kube-apiserver 發生故障，就等於 cluster 的通訊系統癱瘓了，所以 kube-apisever 是一個極為重要的組件。
    
    另外，kube-apiserver 也會隨時監控 Pod 的狀態，這是透過與各個 Node 上的 kubelet 溝通來達成的。

2. **etcd** 

   以 key-value 的方式存放整個 cluster 中的資料，例如 cluster 的狀態、目前存在的資源等等。
   
   備份 etcd 是一項重要的工作，因為當整個 cluster 壞掉時，我們可以藉由還原 etcd 的備分來重建 cluster。

3. **kube-scheduler**

   負責 cluster 中資源的調配，例如哪些 Pod 該放到哪個 Node 上。
   > kube-scheduler 幫 Pod 找到目的 Node 後，scheduler 會通知 kube-apiserver，kube-apiserver 再通知目的 Node 的 kubelet 把 Pod 跑起來。

4. **kube-controller-manager**:

   Cluster 各種物件的管理者，是許多控制器(例如: Node-Controller、Replication-Controller)的集合體，透過 kube-apiserver 監控各種資源，並將資源目前的狀態調整至「期望狀態(Desired status)」。例如 「順利執行」是 Pod 的期望狀態，當 controller-manager 透過 kube-apiserver 發現某個 Pod 壞掉時，controller-manager 會負責重新啟動該 Pod，直到它順利執行。
   
## 小結 --- Kubernetes 的架構與組件

以上就是關於 Node 以及其組件的大致介紹。如果還是覺得有些混亂的話，這裡我們再次用船隊的比喻總結一下：

> 如果想像整個 cluster 是一個船隊，那麼 Node 就是船、Pod 就是船上的貨櫃，容器就是貨櫃裡的貨物、貨物由 Container Runtime 產生。

> 所有的船 (Node) 都有這三個組件：**kubelet**、**kube-proxy**、**Container Runtime**。

> 在船隊中擔任總指揮的主船 (Master Node)，則另外擁有這四個元件 : **kube-apiserver**、**etcd**、**kube-scheduler**、**kube-controller-manager**。

> 總指揮(Master Node)的傳達給小船(Worker Node)的各種指令，都是由 kube-apiserver 發送給船長 kubelet 來完成的。

> 船上的通訊系統由 CNI 負責發 IP、kube-proxy 負責轉發流量。

這裡提供一張圖示：

![https://ithelp.ithome.com.tw/upload/images/20240820/20168692oKw0I7cTyY.png](https://ithelp.ithome.com.tw/upload/images/20240820/20168692oKw0I7cTyY.png)


結合以上介紹，我們來看一下「當使用者告訴 kube-apiserver 要建立一個 Pod 時，會發生了什麼事？」：

1. 使用者透過 CLI 或 API 告訴 api-server 要建立一個 Pod。

2. api-server 對使用者作身分驗證，確認他有建立 Pod 的權限。

3. api-server 將新 Pod 的資訊寫入 etcd。

4. api-server 告訴 scheduler 要找一個合適的 Node 來執行新的 Pod。

5. 找到適合的 Node 後，scheduler 透過 api-server 將該 Node 寫入 etcd。

6. 目前為止，Pod 的 spec 與該去哪裡都僅存在於 etcd 中，還沒有真正的執行。kubelet 透過 api-server 知道有 Pod 要建立，會負責拉取需要的 Image、調用 Container Runtime 建立容器、調用 CNI 來分配 IP 給 Pod。

7. Pod 無論是否成功跑起來，都會向 api-server 回報 Pod 的狀態，狀態會被寫回 etcd。

> 可以發現，任何通訊基本上都要經過 kube-apiserver，這也是為什麼 kube-apiserver 是 cluster 中最重要的組件之一。

## HA Cluster

當 cluster 中僅存在一台 Master Node 時，如果 Master Node 發生故障，雖然在 Worker Node 上的 Pod 仍能正常運作，但是 cluster 的管理功能將會癱瘓，例如：因為缺少在 Master Node 上的 kube-apiserver，管理員就無法用指令或 API 來管理 cluster。

為了避免這種情況，在實際的生產環境中通常會建立一個**高可用性 (High Availability)** 的 cluster，也就是 HA cluster。

所謂「高可用性」就是「提高容錯率」，HA cluster 擁有**多個** Master Node，避免單點故障的情況發生。除此之外，因為多個 Master Node 的原因，etcd 儲存的資料也會有多個副本，這樣一來即使某個 etcd 發生故障，也能透過其他 etcd 的資料來恢復 cluster。

> 不過，多個 Master Node 上就代表著有多個 api-server，因此 HA cluster 會有一個**負載平衡器 (Load Balancer)** 來分配流量。

在設計 HA cluster 時，依據需求的不同，我們有兩種 topology 可以選擇：

* **Stacked etcd topology**

   Stacked etcd topology **至少**需要三台 Master Node 組成。其實我們前面介紹的就是 stacked etcd，也就是 Master Node 與 etcd 部署在同一台機器上，這樣的好處是容易部署與管理，但必須承擔同時失去 Master Node 與 etcd 的風險。

![https://ithelp.ithome.com.tw/upload/images/20240821/201686925hY76eiygC.png](https://ithelp.ithome.com.tw/upload/images/20240821/201686925hY76eiygC.png)

[圖片來源](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

* **External etcd topology**

  External etcd topology **至少**需要三台 Master Node 與三台 etcd server。 將 etcd 獨立部署在另一台機器上，這樣一這樣的好處是可以避免同時失去 Master Node 與 etcd，但部署複雜度較高，且需額外負擔 etcd server 的成本，比起 Stacked etcd 所需的 server 數量多出一倍。
  
![https://ithelp.ithome.com.tw/upload/images/20240821/20168692PsNzn2GdZ3.png](https://ithelp.ithome.com.tw/upload/images/20240821/20168692PsNzn2GdZ3.png)

[圖片來源](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

> 也就是說，「成本與風險」之間的考量決定了我們選擇哪一種 topology。

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