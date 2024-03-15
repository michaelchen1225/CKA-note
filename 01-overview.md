# Day01 Kubernetes簡介

# 序

# 考試比重
Domain | weight
-------|--------
Cluster Architecture, Installation & Configuration |	25%
Workloads & Scheduling	|15%
Services & Networking	|20%
Storage|10%
Troubleshooting	|30%

# 什麼是Kubernetes ?
如果要用一句話介紹Kubernetes，那就是:
>一個容器管理平台

Kubernetes，簡稱`K8s` (因為K到s之間有8個字母)，是一個用於自動化部署、擴展和管理容器化應用程式的**開源**平台。

![https://juststickers.in/wp-content/uploads/2018/11/kubernetes-wordmark.png](https://juststickers.in/wp-content/uploads/2018/11/kubernetes-wordmark.png)

K8s具有以下特點 : 

   - **負載平衡** : 如果對容器的流量很高，K8s能夠實現負載平衡，分發網路流量，從而保持部署的穩定性。

   - **可擴展性** : 可以輕鬆擴展或縮減應用程式的規模，彈性的滿足需求。

   - **容易轉移** : 能快速地將一台機器上的容器轉移到另一台機器上。

   - **自我修復** : 如果容器出現故障，K8s會重新啟動故障的容器，在它們OK以前不會向客戶端開放。

   - **自動部署與回滾** : 你可以描述部署容器時的期望狀態，K8s將以受控的速率將實際狀態更改為期望狀態。例如你可以自動化K8s為`Deployment`部署新的容器，並刪除現有的容器以便將它們的資源移轉到新容器上。

K8s可用於多種應用場景，例如`微服務架構`、`CI/CD`、`雲端部署`等等。

# 今日小節
今天對Kubernetes，也就是`K8s`這個強大的容器管理平台做了簡單的介紹。下一章將會介紹Kubernetes的基本架構及元件，能對Kubernetes的基本運作原理有個大概的認識。

-----



**參考資料**
* [Kubernetes Overview](https://kubernetes.io/docs/concepts/overview/)
* [Kubernetes 官方網站](https://kubernetes.io/)




