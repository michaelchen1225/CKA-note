# 2024 iThome 鐵人賽系列文章：【[入門 Kubernetes 到考取 CKA 證照](https://ithelp.ithome.com.tw/users/20168692/ironman/7376)】


### 前言

讓我們從「雲原生 (Cloud Native)」開始談起。

根據 CNCF(Cloud Native Computing Foundation) 的定義，雲原生旨在讓企業能在公有、私有、混合雲的環境中，建立和執行可擴展的「應用程式」，而這些應用則以**微服務**為架構，以**容器**為基礎在雲端上執行。總之，雲原生並不是一項技術，而是一個「生態」，該生態以容器技術為核心，發展出各式各樣的服務架構與技術，而企業則在雲端環境挑選合適的技術與服務架構，部署穩定、可擴展的應用服務給使用者。

那 CNCF 又是什麼來頭？CNCF 是一個由 Linux 基金會於 2015 年成立的組織，成員包括了 Google、IBM、Microsoft、AWS、Intel 等等，其宗旨為推廣雲原生技術的發展，用一句他們官網上的話來說就是：

> “Make cloud native computing ubiquitous…”  (讓雲原生無所不在)


與 CNCF 一起在 2015 年誕生的，就是 **Kubernetes**。

### 什麼是 Kubernetes？

如果要用一句話介紹 Kubernetes，那就是：

> 容器管理平台

Kubernetes，簡稱「K8s」(因為 K 到 s 之間有 8 個字母)，是一個用於自動化部署、擴展和管理**容器化應用服務**的開源平台。

K8s 的重要特色如下：

   * **自動化部署與回滾**：當應用程式需要更新時，K8s 會「逐步」的更新這些應用，並確保應用程式的運行狀態符合你的期望。如果這些更新有問題，K8s 也能依照你的需求回滾到以前的版本。

   * **負載平衡** : K8s 能將網路流量平均分散到不同的容器上，讓應用服務能即使在流量高的情況下，仍然能穩定的被使用者存取。

   * **自我修復** : 如果容器出現故障，K8s 會嘗試重啟故障的容器，在容器準備好之前不會向使用者開放。

   - **水平擴展性** : 可以輕鬆擴展或縮減應用程式的規模，彈性的調整容器的數量，最佳化雲端算力資源的使用效率。

   - **容易轉移** : 能快速地將一台機器上的應用服務迅速轉移到另一台機器上。


K8s 可用於多種應用場景，例如前面提到的「微服務」，或是 CI/CD 等 DevOps 環境中。

### 鐵人賽文章目標

筆者在前一陣子開始學習 K8s，並且在今年八月份順利通過了 CKA(Certified Kubernetes Administrator) 認證，想藉由本次鐵人賽將當初入門 K8s 的學習筆記整理出來，並且分享 CKA 的考試技巧與心得，希望能夠幫助到 K8s 的初學者與正在準備 CKA 考試的讀者。

### 文章架構與規劃

在開始閱讀文章之前，以下三項技能建議先點起來：

* **Linux 基本操作能力**：cd、ls、chmod、grep、mkdir、tail、curl、systemctl、grep、awk、標準化輸出、管線等等的基本指令與操作就不多提了，重點是要熟悉 vim 的操作方式，例如游標移動、新增行數、回到上一步(undo)、存檔退出等等，因為你將會有許多時間在編輯 yaml，熟悉 vim 能夠讓你輕鬆許多。

* **了解基本的容器概念**：至少要知道容器的基本概念，如果真的沒有概念，這裡提供三個步驟入門：1. 了解「為何需要容器？」、2. 自己打包一個 image、3. 將打包好的 image 變成容器跑起來。這裡提供一個相當不錯的影片教學：「[30 分鐘 Docker 入門教程](https://www.youtube.com/watch?v=Ozb9mZg7MVM)」。

* **網路的基本概念**：例如 IP、Port、DNS、路由規則等等，可以去網路上找一篇講網路概論的文章補齊，簡單了解即可。

> 以上三點每項的學習成本大約 0.5 ~ 1.5 小時左右，就是了解最基本的即可~

系列的開頭將會從 K8s 的基礎概念開始介紹，中間則會以 CKA 五大考試領域為章節，搭配實作來介紹不同的重點概念，最後在結尾分享 CKA 的考試技巧與心得，另外也將提供附錄做額外補充。

因此，章節的劃分大致規劃如下：

> 註：「*」為 *CKA Optional*，如果是專攻 CKA 的讀者，可以先跳過這個部分，後面有興趣再回來看(例如 helm)。

1. **Basic concept**：K8s 基本的概念，例如 Pod、Deployment、Service、Namespace、Label、Rolling update 等等。目前本章已更新完畢：

| 天數 | 主題|
| --- | --- |
| Day 02 |[Kubernetes 的架構與組件](https://ithelp.ithome.com.tw/articles/10345505)
| Day 03 |[使用 Kubeadm 建立 Kubernetes Cluster + 相關的 Bonus Tips](https://ithelp.ithome.com.tw/articles/10345660)
| Day 04 |[Pod](https://ithelp.ithome.com.tw/articles/10345796)
| Day 05 |[Pod 中的環境變數與指令](https://ithelp.ithome.com.tw/articles/10345967)
| Day 06 |[ReplicaSet、Deployment & StatefulSet](https://ithelp.ithome.com.tw/articles/10346089)
| Day 07 |[Rolling Update & Rollback](https://ithelp.ithome.com.tw/articles/10346223)
| Day 08 |[Namespace](https://ithelp.ithome.com.tw/articles/10346374)
| Day 09 |[Service](https://ithelp.ithome.com.tw/articles/10346530)
| Day 10 |[kubectl 基本操作彙整](https://ithelp.ithome.com.tw/articles/10346691)
| Day 11 |[*好用的專案部署工具 --- Helm](https://ithelp.ithome.com.tw/articles/10346850)

2. **Storage**：K8s 中的儲存概念，例如 ConfigMap、Secret、Volume、PV、PVC、StorageClass。目前本章已更新完畢：

| 天數 | 主題|
| --- | --- |
| Day 12 |[ConfigMap & Secret](https://ithelp.ithome.com.tw/articles/10347004)
| Day 13 |[Volume 的三種基本應用 --- emptyDir、hostPath、configMap & secret](https://ithelp.ithome.com.tw/articles/10347182)
| Day 14 |[PV、PVC & StorageClass](https://ithelp.ithome.com.tw/articles/10347335)


3. **Workloads & Scheduling**：K8s 中有工作附載、調度等概念，例如資源分配、Pod scheduling 的策略、Deployment 的部署策略。目前更新：

| 天數 | 主題|
| --- | --- |
| Day 15 |[Manual Scheduling(上)：nodeName & nodeSelector](https://ithelp.ithome.com.tw/articles/10347495)
| Day 16 |[Manual Scheduling(下)：Affinity & Taint](https://ithelp.ithome.com.tw/articles/10347661)
| Day 17 |[Static Pod & DaemonSet](https://ithelp.ithome.com.tw/articles/10347876)
| Day 18 |[*進階部署策略：Blue-Green & Canary](https://ithelp.ithome.com.tw/articles/10348066)
| Day 19 |[資源管理 --- Pod QoS、LimitRange & ResourceQuota](https://ithelp.ithome.com.tw/articles/10348214)


4. **Services & Networking**：K8s 中的基本網路概念以及應用，例如 TLS 管理、Ingress、Network Policy。

| 天數 | 主題|
| --- | --- |
| Day 20 |[Kubernetes 的網路基本架構](https://ithelp.ithome.com.tw/articles/10348418)
| Day 21 |[TLS/SSL in Kubernetes](https://ithelp.ithome.com.tw/articles/10348555)
| Day 22 |[憑證管理與kubeconfig](https://ithelp.ithome.com.tw/articles/10348787)

5. **Cluster Architecture, Installation & Configuration**：K8s cluster 的基本設定，例如備份、升級 cluster、權限管理等等。

6. **Troubleshooting**：K8s 中的故障排除以及相關監控方式。

在文章的內容方面，系列的章節雖然以 CKA 的考試領域來劃分，但仍會提及 CKA 考試範圍之外的概念與相關實作，希望能盡可能涵蓋到 K8s 的基礎概念。

而文章的具體架構大致上會先介紹今天的學習目標，再進行概念介紹與實作，最後進行總結。

### CKA 簡介

目前 CNCF 比較重要的 K8s 相關認證有以下三張：

* CKA
* CKAD
* CKS

> CKA 偏向管理者，CKAD 偏向開發人員，而 CKS 則偏向資安人員。

CKA 考試均為實作題，15~20 題不等，需要在兩小時內寫完，而上面提到的考試五大領域如下：

Domain | weight
-------|--------
Storage | 10%
Troubleshooting | 30%
Workloads & Scheduling | 15%
Cluster Architecture, Installation & Configuration | 25%
Services & Networking | 20%


> 五大領域下面還有更細的子領域，可參考 [CKA 官網](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/#)

CKA 考試的及格分數為 66 分，而筆者在今年八月順利通過了考試：

**考試成績：**

![https://ithelp.ithome.com.tw/upload/images/20240819/20168692U2t6Q9lv6q.png](https://ithelp.ithome.com.tw/upload/images/20240819/20168692U2t6Q9lv6q.png)

**證書：**

![https://ithelp.ithome.com.tw/upload/images/20240819/20168692BoTtjh8w8g.png](https://ithelp.ithome.com.tw/upload/images/20240819/20168692BoTtjh8w8g.png)

### 今日小節

今天我們從雲原生談起，介紹了 CNCF 與 Kubernetes 的由來，並且簡單介紹了 Kubernetes 的特色，最後提到了本次鐵人賽的文章規劃與目標，希望透過基礎概念的介紹與實作，幫助到想入門 Kubernetes 或正在準備 CKA 的讀者！

---
**參考資料**

* [CNCF official website](https://www.cncf.io/)

* [CNCF Wikipedia](https://en.wikipedia.org/wiki/Cloud_Native_Computing_Foundation)

* [Kubernetes official website](https://kubernetes.io/)

* [Certified Kubernetes Administrator (CKA)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)

* [什麼是雲端原生？](https://aws.amazon.com/tw/what-is/cloud-native/)

* [【科技云原声】第1期-何谓云原生？如何走近云原生？](https://www.youtube.com/watch?v=LIxvSwCOazI)