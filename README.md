> [章節列表](#章節規劃)

## 用 CKA 入門 Kubernetes

這個系列共 31 篇文章，主要聚焦於 CKA 的考試範圍。非考試範圍的內容可以參考另一個 repo：[Kubernetes notes](https://github.com/michaelchen1225/Kubernetes-note)

文章內容會假設讀者從來沒有碰過 K8s，內容都是非常基礎的概念與操作，開頭將會從 K8s 的基礎概念開始介紹，中間則會以 CKA 五大考試領域為章節，搭配實作來介紹不同的操作應用，最後在附錄分享 [CKA 的考試技巧與心得](./附錄/CKA%20攻略：我是如何滿分一次通過的？.md)。

### 前言

讓我們從「雲原生 (Cloud Native)」開始談起。

根據 CNCF(Cloud Native Computing Foundation) 的定義，雲原生旨在讓企業能在公有、私有、混合雲的環境中，建立和執行可擴展的「應用程式」，而這些應用則以**微服務**為架構，以**容器**為基礎在雲端上執行。
  
總之，雲原生並不是一項技術，而是一個「生態」，該生態以容器技術為核心，發展出各式各樣的服務架構與技術，而企業則在雲端環境挑選合適的技術與服務架構，部署穩定、可擴展的應用服務給使用者。

那為什麼最近雲原生變成了一個熱門的「buzzword」呢？這就得談談伺服器與服務架構的演進了。

## 從實體機到容器

我們知道用程式碼把應用服務寫好後，會跑在伺服器上執行讓使用者存取。

而伺服器其實就是一台「**實體電腦**」，但如果不同的服務都跑在同一台實體伺服器上，可能會造成套件、版本、互搶資源等等的衝突。但為了隔離程式而購買多台實體機顯然不划算，因此有了「**虛擬機**」的出現。

在實體電腦上模擬出多個虛擬機，進而依需求對程式進行隔離，且在具備隔離性的同時，執行環境的遷移也比以往方便。不過虛擬機畢竟還是模擬了一整台電腦，對於服務來說有些功能根本用不到，所以虛擬機還是顯得有些臃腫。所以後來就出現了**容器**。

容器在具備「**高隔離性**」與「**環境轉移方便**」的同時，遠比虛擬機來的更加**輕量**，因為容器只需要打包「服務本身」與其「相依的執行環境」，而不需要包含整個作業系統，所以不像虛擬機那麼肥，還能做到「**執行環境的統一**」，避免了過去伺服器、虛擬機時代，因為套件、版本差異而出現「在工程師筆電上能跑，拿到伺服器上就跑不動」的情況。

> 以演進的方向來看，服務的執行環境正在朝著「輕量、獨立」的方向前進。

![https://ithelp.ithome.com.tw/upload/images/20250224/20168692KYGMlcIIIi.png](https://ithelp.ithome.com.tw/upload/images/20250224/20168692KYGMlcIIIi.png)

> [圖片來源](https://kubernetes.io/docs/concepts/overview/)

## 從單體式到微服務

而容器的出現，則讓微服務架構的實踐更加如魚得水。

在以前單體式架構的時代，所有功能都寫在同一個單元，隨然開發上較方便且直覺，但一旦某個小功能出錯可能會牽連到整個服務，且除錯、新增或刪除功能、版本更新時除了會影響整個服務之外，光是單元測試就要花不少時間。

而微服務的設計理念則是：

> 將服務中的每個小功能**獨立**出來，每個小功能彼此之間互相溝通、配合，對外看起來是一個服務整體，實際上則可以獨立的撰寫、除錯、部署、更新，在新增/刪除功能時也不用擔心影響到其他功能與整體服務。

![https://ithelp.ithome.com.tw/upload/images/20250224/20168692U55D1dCLNp.png](https://ithelp.ithome.com.tw/upload/images/20250224/20168692U55D1dCLNp.png)

> [圖片來源](https://microservices.io/patterns/microservices.html)

容器的出現正好與微服務的設計理念不謀而合，將小功能打包成獨立的容器，完美的實現了微服務之間的「獨立性」，再搭配**雲端平台**各種的部署、監控服務，「雲原生」就成為了現在大家常聽到的熱門詞彙了。

講了那麼多，那剛開始提到的 CNCF 又是什麼來頭？CNCF 是一個由 Linux 基金會於 2015 年成立的組織，成員包括了 Google、IBM、Microsoft、AWS、Intel 等等，其宗旨為推廣雲原生技術的發展，用一句他們官網上的話來說就是：

> “Make cloud native computing ubiquitous…”  (讓雲原生無所不在)

但隨著容器、微服務架構、雲原生概念的興起，容器的使用數量也越來越多、規模也越加龐大。在傳統容器技術的架構下，管理大量的容器會面臨一些挑戰與問題，例如：

* 負載問題：以 Docker 為例，容器都是跑在「有安裝 Docker」的**單一主機**上，但單一主機無法負荷大量的容器。

* 跨主機通訊：雖然可以用多台主機把大量的容器跑起來(分散式系統)，但不同主機上的容器通訊就不再像從前單一主機上那樣單純，需要額外管理。

* 容器調度：在分散式系統的架構下，該如何選擇資源充足的主機來部署新容器？若某台主機的資源已經不夠用了，該如何將上面的部分容器遷移至其他主機？

* 版本更新：當容器中的 image 需要更新時，如何在不影響使用者的前提下，有效率的完成更新(ex. zero downtime)？

* 容器監控：如何有效的監控所有主機、容器的狀況？若發現容器壞了該如何復原？

上述的問題的可說是分散式架構中，容器化應用的痛點。

核心問題在於，傳統容器引擎只負責「單一主機」上的容器，開發人員就得自行協調、設定、管理不同主機上的大量容器，管理難度相當高。這時，如果能有一個「一站式」的容器管理平台該有多好？

因此在 2015 年，**Kubernetes** 誕生了。

## 什麼是 Kubernetes？

如果用一句話介紹 Kubernetes，就是：**容器編排(orchestration)系統**。

「Orchestration」可能乍看之下有點抽象，這裡以「交響曲」樂隊來舉例說明：

* 樂隊的演出時，觀眾的聽感為一個整體，也就是交響曲，例如「貝多芬的第七號交響曲」。
* 在同一首交響曲中，會使用到不同的樂器，不同的樂器要負責的 part 也不同，所以各樂器都會自己的「樂譜」。
* 樂隊會由指揮家來領導，指揮家得先理解交響樂的整體編排架構，才能指揮不同樂器和諧的演出，最終帶給觀眾一個完整、美好的聽覺體驗。
* 而這些都來源於貝多芬寫曲時的「編排」。

上述「交響曲」的例子可以這樣對應到「容器化應用」的場景中：

* 貝多芬：開發人員
* 第七號交響曲：提供給使用者的服務
* 各類樂器：提供不同功能的容器
* 各類樂譜：不同容器的 image
* 指揮家：Kubernetes

> 如果沒有 Kubernetes，就好像貝多芬沒有指揮家，就又得編曲又得指揮，光想就很累人。

換到應用服務的場景來說，如果開發人員對於整體的容器編排(ex.容器定義、排程、調度、監控、安全性管理、資源分配、網路配置、故障處理...)已經做好規劃，只需要針對「容器編排(container orchestration)系統」做一些設定，就能**自動**完成這些目標，進而更有效率的提供使用者該有的服務。


> Container orchestration 的核心目標：自動的「維持」開發者規劃好的期望狀態。

總而言之，Kubernetes 是一個開源的**容器編排(orchestration)系統**，用於部署、擴展和管理「容器化應用服務(containerized applications)」。

在目前微服務的架構下，可能有成百上千個容器，而這麼多容器該如何有效的管理，在目前 Kubernetes 就是最主流的解決方案。又因為 K 到 s 之間有 8 個字母，Kubernetes 常被簡稱為 K8s。

K8s 的能夠做到以下幾點：

  * **自動化部署與回滾**：當應用程式需要更新時，K8s 會「逐步」的更新這些應用，並確保應用程式的運行狀態符合你的期望。如果這些更新有問題，K8s 也能依照你的需求回滾到以前的版本。

  * **負載平衡** : K8s 能將網路流量平均分散到不同的容器上，讓應用服務能即使在流量高的情況下，仍然能穩定的被使用者存取。

  * **自我修復** : 如果容器出現故障，K8s 會嘗試重啟故障的容器，在容器準備好之前不會向使用者開放。

  * **水平擴展性** : 可以輕鬆擴展或縮減應用程式的規模，彈性的調整容器的數量，最佳化基礎設施資源的使用效率。

  * **跨主機的容器編排**：上述的特點其實不一定需要 K8s 就能做到，例如 docker 搭配 Autoscaler 就能做到自動擴展容器。「編排(orchestration)」可謂是 K8s 的精隨，從容器化應用的定義、排程、調度、監控、安全性管理、資源分配等等都能在 K8s 上設定，只要設定好了後面就交給 K8s 自動完成。另外 K8s 本身為分散式架構，特別適合處理容器跑在多台主機上的場景(只需針對 K8s 設定，而非跑到個別主機上手動管理)。
  
另外， K8s 有一個重要特點，就是「開源」。

K8s 累積了相當龐大且活躍的開源生態，其中各種的插件與應用涵蓋了各式各樣的範圍，例如偏向底層的「Container Network Interface Plugin」，到「監控(Monitoring)」的各種 solution，使用者都能自行挑選。

在應用場景上，Kubernetes 可用於前面提到的「微服務」，或是 CI/CD 等 DevOps 環境中。雖然 Kuberentes 起源於雲原生，但也可以部署在本地環境中，在部署環境的選擇上也有相當的彈性。

從 plugin、solution、部署環境的高彈性選擇中不難看出，K8s 不會被某個雲端平台綁死，在轉換部署環境的陣痛期會比較短，這或許也是主流雲端平台都會提供 K8s 服務的原因之一。

## 用 CKA 入門 Kubernetes

CKA 是一張證照，全稱為「Certified Kubernetes Administrator」，針對「管理 Kubernetes」進行考核。

雖然 Kubernetes 是一個超級深水區，但 CKA 的考試範圍都是 K8s 的基本操作，因此，「考取 CKA」並不代表精通了 K8s，但卻是一個「入門 K8s」的不錯選擇，因為在準備考試的過程中，將會學習到許多 K8s 的基本概念與操作，為後續較為進階的操作與應用打下穩固的基礎。

> 考證照是需要花錢報名考試的，但如果以「打基礎」的角度來看，並不一定要花錢去考照，重點是具備「能」考過 CKA 的基礎知識。

### 文章架構與規劃

在開始閱讀文章之前，以下三項技能建議先點起來：

* **Linux 基本操作能力**：cd、ls、chmod、grep、mkdir、tail、curl、systemctl、標準化輸出、管線等等的操作就不多提了，重點是熟悉 vim 的操作方式，例如游標移動、新增行數、回到上一步(undo)、存檔退出等等，因為之後會有許多時間在編輯 yaml，熟悉 vim 會方便許多。

* **了解最基本的容器概念**：至少要知道容器的基本概念，如果真的沒有概念，這裡提供三個步驟入門：
  1. 了解「為何需要容器？」
  2. 自己打包一個 image
  3. 將打包好的 image 變成容器跑起來。

  > 這裡提供一個相當不錯的影片教學：「[30 分鐘 Docker 入門教程](https://www.youtube.com/watch?v=Ozb9mZg7MVM)」。

* **網路的基本概念**：例如 IP、Port、DNS、路由規則等等，可以去網路上找一篇講網路概論的文章補齊，簡單了解就好。

### 章節規劃

1. **Basic concept**：

| 天數 | 主題|
| --- | --- |
| Day 02 |[Kubernetes 的架構與組件](CH1-to-6/01-Basic-Concept/Day%2002%20-【Basic%20Concept】：Kubernetes%20的架構與組件.md)
| Day 03 |[建立 Kubeadm Cluster +  Bonus Tips](CH1-to-6/01-Basic-Concept/Day%2003%20-【Basic%20Concept】：使用%20Kubeadm%20建立%20Kubernetes%20Cluster%20+%20相關的%20Bonus%20Tips.md)
| Day 04 |[Pod](CH1-to-6/01-Basic-Concept/Day%2004%20-【Basic%20Concept】：Pod.md)
| Day 05 |[Pod 中的環境變數與指令](CH1-to-6/01-Basic-Concept/Day%2005%20-【Basic%20Concept】：Pod%20中的環境變數與指令.md)
| Day 06 |[ReplicaSet、Deployment & StatefulSet](CH1-to-6/01-Basic-Concept/Day%2006%20-【Basic%20Concept】：ReplicaSet、Deployment%20&%20StatefulSet.md)
| Day 07 |[Rolling Update & Rollback](CH1-to-6/01-Basic-Concept/Day%2007%20-【Basic%20Concept】：Rolling%20Update%20&%20Rollback.md)
| Day 08 |[Namespace](CH1-to-6/01-Basic-Concept/Day%2008%20-【Basic%20Concept】：Namespace.md)
| Day 09 |[Service](CH1-to-6/01-Basic-Concept/Day%2009%20-【Basic%20Concept】：Service.md)
| Day 10 |[kubectl 基本操作彙整](CH1-to-6/01-Basic-Concept/Day%2010%20-【Basic%20Concept】：kubectl%20基本指令彙整.md)
| Day 11 |[專案的打包與部署 --- Helm](CH1-to-6/01-Basic-Concept/Day%2011%20-【Basic%20Concept】：Helm.md)

2. **Storage**：

| 天數 | 主題|
| --- | --- |
| Day 12 |[ConfigMap & Secret](CH1-to-6/02-Storage/Day%2012%20-【Storage】：ConfigMap%20&%20Secret.md)
| Day 13 |[Volume 的三種基本應用 --- emptyDir、hostPath、configMap & secret](CH1-to-6/02-Storage/Day%2013%20-【Storage】：Volume%20的三種基本應用%20---%20emptyDir、hostPath、configMap%20&%20secret.md)
| Day 14 |[PV、PVC & StorageClass](CH1-to-6/02-Storage/Day%2014%20-【Storage】：%20PV、PVC%20&%20StorageClass.md)


3. **Workloads & Scheduling**：

| 天數 | 主題|
| --- | --- |
| Day 15 |[Manual Scheduling(上)：nodeName & nodeSelector](CH1-to-6/03-Workloads-and-Scheduling/Day%2015%20-【Workloads%20&%20Scheduling】：Manual%20Scheduling(上)：nodeName%20&%20nodeSelector.md)
| Day 16 |[Manual Scheduling(下)：Affinity & Taint](CH1-to-6/03-Workloads-and-Scheduling/Day%2016%20-【Workloads%20&%20Scheduling】：Manual%20Scheduling(下)：Affinity%20&%20Taint.md.md)
| Day 17 |[Static Pod & DaemonSet](CH1-to-6/03-Workloads-and-Scheduling/Day%2017-【Workloads%20&%20Scheduling】：Static%20Pod%20&%20DaemonSet.md)
| Day 18 |[*進階部署策略：Blue-Green & Canary](CH1-to-6/03-Workloads-and-Scheduling/Day%2018-【Workloads%20&%20Scheduling】：進階部署策略：Blue%20Green%20&%20Canary.md)
| Day 19 |[資源管理 --- Pod QoS、LimitRange & ResourceQuota](CH1-to-6/03-Workloads-and-Scheduling/Day%2019-【Workloads%20&%20Scheduling】：資源管理%20---%20Pod%20QoS、LimitRange%20&%20ResourceQuota.md)

4. **Services & Networking**：

| 天數 | 主題|
| --- | --- |
| Day 20 |[Kubernetes 的網路基本架構](CH1-to-6/04-Services-and-Networking/Day%2020%20-【Services%20&%20Networking】：Kubernetes%20的網路基本架構.md)
| Day 21 |[TLS/SSL in Kubernetes](CH1-to-6/04-Services-and-Networking/Day%2021%20-【Services%20&%20Networking】：TLS%20in%20Kubernetes.md)
| Day 22 |[憑證管理與kubeconfig](CH1-to-6/04-Services-and-Networking/Day%2022%20-【Services%20&%20Networking】：憑證管理與kubeconfig.md)
| Day 23 |[Service 的路由 --- Ingress](CH1-to-6/04-Services-and-Networking/Day%2023%20-【Services%20&%20Networking】：Service%20的路由%20---%20Ingress.md)
| Day 24 |[Pod 的守門員 --- Network Policy](CH1-to-6/04-Services-and-Networking/Day%2024%20-【Services%20&%20Networking】：網路連線的守門員%20---%20Network%20Policy.md)

5. **Cluster Configuration**：

| 天數 | 主題|
| --- | --- |
| Day 25 |[Cluster upgrade](CH1-to-6/05-Cluster-Architecture-Installation-and-Configuration/Day%2025%20-%20【Cluster%20Configuration】：Cluster%20upgrade.md)
| Day 26 |[etcd 的備份與還原](CH1-to-6/05-Cluster-Architecture-Installation-and-Configuration/Day%2026%20-%20【Cluster%20Configuration】：etcd%20的備份與還原.md)
| Day 27 |[權限管理 (一)：RBAC](CH1-to-6/05-Cluster-Architecture-Installation-and-Configuration/Day%2027%20-%20【Cluster%20Configuration】：權限管理%20(一)：RBAC.md)
| Day 28 |[權限管理 (二)：Service Account](CH1-to-6/05-Cluster-Architecture-Installation-and-Configuration/Day%2028%20-%20【Cluster%20Configuration】：權限管理%20(二)：Service%20Account.md)
| Day 29 |[權限管理 (三)：Security Context](CH1-to-6/05-Cluster-Architecture-Installation-and-Configuration/Day%2029%20-%20【Cluster%20Configuration】：權限管理%20(三)：Security%20Context.md)

6. **Troubleshooting & Monitoring**：

| 天數 | 主題|
| --- | --- |
| Day 30 |[Pod 的生命週期與監控](CH1-to-6/06-Troubleshooting-and-Monitoring/Day%2030%20-【Monitoring】：Pod%20的命週期與監控.md)
| Day 31 |[Troubleshooting 小技巧](CH1-to-6/06-Troubleshooting-and-Monitoring/Day%2031%20-【Troubleshooting】：Troubleshooting%20小技巧.md)

在文章的內容方面，系列的章節雖然以 CKA 的考試領域來劃分，但仍會提及 CKA 考試範圍之外的概念與相關實作，盡可能的涵蓋 K8s 的基礎操作。

## CKA 簡介

目前 CNCF 比較重要的 K8s 相關認證有以下三張：

* CKA
* CKAD
* CKS

> CKA 主要針對 Kubernetes 的操作與管理能力進行考核，較偏向「管理」，CKAD 則偏向「開發」，而 CKS 偏向「資安」。

CKA 考試均為實作題，15~20 題不等，需要在兩小時內寫完，考試範圍有五大領域：

Domain | weight
-------|--------
Storage | 10%
Troubleshooting | 30%
Workloads & Scheduling | 15%
Cluster Architecture, Installation & Configuration | 25%
Services & Networking | 20%


> 五大領域下面還有更細的子領域，可參考 [CKA 官網](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/#)

### **重要提醒**：CKA 2025 改版

最近(2025/4)筆者在 Reddit 上閒逛的時候，發現有篇文章討論了 CKA 考試內容的更新，原文連結在[這裡](https://www.reddit.com/r/CKAExam/comments/1jbi4iw/discussion_of_the_updated_feb_18th_2025_cka_exam/)，以下大概說明一下目前的改動有哪些：

* 增加新的考試主題，例如 Helm、Kustomize、CRD、Gateway API、HPA 等等。
* 考試難度增加，以前更多的是找文件上的 YAML 改改，現在更注重故障排除的能力。
* 移除了一些必考題，例如 ETCD 的備份與還原。
* 根據一些考生的留言，考試出現了他們在準備時沒看過、不熟的東西，例如使用 dpkg、CRD 等等。

關於 Helm 與 Kustomize 的部分，筆者已經更新於鐵人賽的文章中，不過其他的部分就只能等到之後有空再來更新了。如果你是 2025/2/18 CKA 改版後的考生，請務必留意這些更動內容，Google 搜尋「 Updated Feb 18th 2025 CKA Exam」就能找到相關的討論與資訊。

總而言之，新的考試題目與筆者在去年(2024)八月份已經截然不同，請考生要額外留意以下主題：

* Helm & Kustomize
* CRD
* Gateway API
* HPA、VPA
* Gateway API
* 安裝 Container Runtime

好消息是報名考試後的模擬考 Killer.sh 也更新了他們的考題，所以還是能拿模擬考的分數來衡量自己是否準備好了。

---

CKA 考試的及格分數為 66 分，筆者在今年八月順利通過了考試：

> [我的 CKA 攻略](https://ithelp.ithome.com.tw/articles/10354313)

**考試成績：**

![https://ithelp.ithome.com.tw/upload/images/20240819/20168692U2t6Q9lv6q.png](https://ithelp.ithome.com.tw/upload/images/20240819/20168692U2t6Q9lv6q.png)

**證書：**

![https://ithelp.ithome.com.tw/upload/images/20240819/20168692BoTtjh8w8g.png](https://ithelp.ithome.com.tw/upload/images/20240819/20168692BoTtjh8w8g.png)

## 今日小節

今天我們從雲原生談起，從容器、微服務談到了 Kubernetes，簡單介紹了 Kubernetes 的特色，其中最重要的特色就是「彈性」，從容器編排、插件、部署環境等等，我們都能自行選擇、規劃。

今天的最後提到了本次鐵人賽的文章規劃與目標，希望透過基礎概念的介紹與實作，幫助到想入門 Kubernetes 或正在準備 CKA 的讀者！

---
**參考資料**

* [CNCF official website](https://www.cncf.io/)

* [CNCF Wikipedia](https://en.wikipedia.org/wiki/Cloud_Native_Computing_Foundation)

* [Kubernetes official website](https://kubernetes.io/)

* [Certified Kubernetes Administrator (CKA)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)

* [什麼是雲端原生？](https://aws.amazon.com/tw/what-is/cloud-native/)

* [【科技云原声】第1期-何谓云原生？如何走近云原生？](https://www.youtube.com/watch?v=LIxvSwCOazI)
