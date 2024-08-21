# Day 03 - Basic Concept：使用 Kubeadm 建立 Kubernetes Cluster + 相關的 Bonus Tips

### 今日目標

* 準備一個 Kubernetes cluster 做為練習環境

* Bonus Tips：
  * 設定 kubectl bash completion
  * 在 Worker Node 使用 kubectl
  * kubeadm 初始化過程中的除錯
  * 建立 single node cluster
  * 移除 cluster 中的 Worker Node


在接下來的章節中，將會有許多範例或練習需要對 cluster 進行操作，所以需要先準備練習用的 cluster，以下提供了兩種方法進行建置：

### 方法一 : playground

這是最簡單輕鬆的方式，直接使用網路上的 playground 進行練習，不需要額外的環境建置。這邊提供了兩個網站，登入後就能得到一個*暫時性*的 cluster，可以進行練習：

  1. [killercoda](https://killercoda.com/)
  2. [Play with Kubernetes](https://labs.play-with-k8s.com/)

兩者的主要差別是，killercoda 開一個環境比較方便，但最多只有兩個 Node 可以用。
如果需要三個以上的 Node 可以使用 Play with Kubernetes。

### 方法二 : kubeadm

但使用 playground 的方式，練習的結果是暫時性的。所以如果想要建立一個較為完整的 cluster，可以使用「kubeadm」進行建置。

kubeadm 是一個專門用來部署 Kubernetes 的工具，能夠快速的建立一個 cluster，並且可以直接在本地端進行操作。以下將以 Virtualbox 為例，用 kubeadm 建立一個 cluster。

使用 kubeadm 進行建置 cluster 的步驟如下：

  1. 準備環境(虛擬機)
  2. 安裝 container runtime
  3. 安裝必要組件：kubelet、kubeadm、kubectl
  4. 關閉 swap 並啟用 ip_forward
  5. 初始化 master node
  6. 加入 worker node
  7. 安裝 Pod network

### STEP 1：準備環境

首先需要安裝 Virtualbox (當然你也可以選用其他虛擬機平台)，並建立至少兩台的 VM 做為 Node 來組成 cluster。

其中一台 VN 作為 Master Node，其餘的作為 Worker Node。需注意的是，每台 VM 只少需要：

  * 2GB RAM
  * 2 CPU

作業系統這裡挑選 Ubuntu 22.04。在網路設定方面，每台虛擬機準備兩張網路卡：

  * 一張選用 NAT
  * 一張選用橋接介面卡，方便 cluster 內部的 VM 溝通。記得這張網路卡不要用 DHCP，需要自行手動設定 IP。例如下表：
  
  VM | IP
  ---|---
  Master | 192.168.132.1
  Worker1 | 192.168.132.2
  Worker2 | 192.168.132.3

至於安裝 VM 與設定 IP 的方式就不再此贅述，可以參考以下文章：

  * [在virtualbox上安裝Ubuntu](https://karenkaods.medium.com/%E4%B8%89%E6%AD%A5%E9%A9%9F%E5%9C%A8-windows-%E9%9B%BB%E8%85%A6%E4%B8%8A%E5%AE%89%E8%A3%9D-vitrualbox-%E5%95%9F%E5%8B%95-ubuntu-%E8%99%9B%E6%93%AC%E6%A9%9F-f45619d3c088)
  
  * [設定Ubuntu IP](https://sam.liho.tw/2022/09/29/ubuntu-22-04-%E6%8C%87%E4%BB%A4-cli-%E8%A8%AD%E5%AE%9A%E7%B6%B2%E8%B7%AF%E7%AD%86%E8%A8%98/)

安裝好虛擬機和設定好 IP 後，就可以開始 cluster 的建置了。

---

**Tips：Single node cluster**

其實只用一台 VM 即可建立 cluster，這樣的方式稱為「single node cluster」。

如果你手頭上的資源沒有很多，就可以考慮建立 single node cluster 來當作練習環境，同樣按照下面的步驟進行建置，不過要注意做完「STEP 5」後，**不須操作**「*STEP 6 : 加入worker node*」，直接跳到「*STEP 7 : 安裝 Pod network*」，最後記得看「Tips 4: Single node cluster」。

> 記不住沒關係，底下如果遇到需要 single node cluster 的情況，會再次提醒！


### STEP 2：安裝 container runtime

**每台** VM 上都需要安裝 container runtime，這裡以 containerd 為例：

> 不同 Linux 版本的安裝步驟可以參考[官方文件](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf)，這裡以 Ubuntu 為例。

設定好 Apt repo 的 GPG key：

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

加入 containerd 的 repo 到 Apt 的 source 中：

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

安裝 containerd：

```bash
sudo apt-get install -y containerd.io
```

安裝後檢查一下是否有正常運作:
```bash
sudo systemctl enable containerd
systemctl status containerd
```

接著安裝 crictl，一個用來操作 Container Runtime Interface (CRI) 的 CLI 工具：

```bash
VERSION="v1.30.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

設定 crictl 需要的 socket 位置：

```bash
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

> **補充**：containerd 和 Docker 都是 container runtime， 兩者的差異可以參考[這裡](https://cloud.tencent.com/document/product/457/35747)

安裝 containerd 與 crictl 後，需要將 cgroup-driver 設定為 k8s cluster 所需的 **systemd**：

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

編輯 /etc/containerd/config.toml，找到「[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]」，將 SystemdCgroup 設定為 true：

```yaml
......
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  BinaryName = ""
  CriuImagePath = ""
  CriuPath = ""
  CriuWorkPath = ""
  IoGid = 0
  IoUid = 0
  NoNewKeyring = false
  NoPivotRoot = false
  Root = ""
  ShimCgroup = ""
  SystemdCgroup = true # 改這裡!
......
```

重新啟動 containerd：
```bash
sudo systemctl restart containerd
```

最終檢查一下是否有將「SystemdCgroup=ture」設定成功：
```bash
containerd config dump | grep SystemdCgroup
# SystemdCgroup = true
```

**提醒**

請確保**每台** VM 都安裝了 container runtime，並且 SystemCgroup 也有設定為 true，否則 cluster 建立後重要元件會不斷重啟而無法使用！

### STEP 3：安裝必要組件: kubelet、kubeadm、kubectl

在每台 VM 上，需要以下三個組件：

  * **kubelet**：昨天的文章中有提到，它是「小船的船長」
  * **kubeadm**：用來部署 cluster 的工具
  * **kubectl**：管理員用來與 cluster 進行溝通的 CLI 工具，讓你能透過下指令的方式操作 cluster

安裝以上三個組件的方式如下：

* 檢查有沒有 /etc/opt/keyrings 這個目錄，如果沒有的話先建立起來：

```bash
ls -d /etc/apt/keyrings 2> /dev/null || sudo mkdir -p -m 755 /etc/apt/keyrings
```

* 首先，把 kubernetes 的 repo 加入到 apt 的 source list 中：
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

> 如果作業系統是其他版本的 Linux，可參考[官方文件](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) 

* 查看目前可用的 kubeadm 版本：
```bash
sudo apt update
sudo apt-cache madison kubeadm
# 這裡會列出可用的版本，以下範例選用 1.31.0-1.1
```

* 安裝 kubelet、kubeadm、kubectl

```bash
sudo apt-get update
sudo apt-get install -y kubelet=1.31.0-1.1 kubeadm=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

* 檢查 kubeadm 是否安裝成功 

```bash
kubeadm version
```
輸出：
```text
kubeadm version: &version.Info{Major:"1", Minor:"31", GitVersion:"v1.31.0", GitCommit:"9edcffcde5595e8a5b1a35f88c421764e575afce", GitTreeState:"clean", BuildDate:"2024-08-13T07:35:57Z", GoVersion:"go1.22.5", Compiler:"gc", Platform:"linux/amd64"}
```

* 檢查 kubelet 是否安裝成功

```bash
kubelet --version
```
輸出：
```text
Kubernetes v1.31.0
```

* 檢查 kubectl 是否安裝成功

```bash
kubectl version --client
```
輸出：
```text
Client Version: v1.31.0
Kustomize Version: v5.4.2
```

> 確認每台 VM 都成功安裝了 kubeadm、kubelet、kubectl 後，再往下進行步驟四。

### STEP4：關閉 swap 並啟用 ip_forward

在預設上，如果 swap 沒有被關閉，可能會導致 kubelet 無法正常運作。所以需要先關閉所有 VM 上的 swap：

```bash
sudo swapoff -a # 暫時關閉
sudo vim /etc/fstab # 永久關閉，將swap的那一行註解掉
```

* 每台 VM 都需要載入必要的模組，並啟用 ip_forward：

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo echo -e  'overlay\nbr_netfilter' | sudo tee /etc/modules-load.d/containerd.conf
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

* 確認模組是否載入成功:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

* 確認「net.bridge.bridge-nf-call-ip6tables」、「net.bridge.bridge-nf-call-iptables」、「net.ipv4.ip_forward」都是1:

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

> 確認每台 VM 都成功關閉 swap、模組有成功載入，並啟用 ip_forward 後，再往下進行步驟五。

### STEP 5：初始化 Master Node

初始化時，記得指定 kube-apiserver 所在的 Master Node IP：

> 以下操作僅於 Master Node 上操作：

```bash
sudo kubeadm init --apiserver-advertise-address <master node IP> --control-plane-endpoint <master node IP> --pod-network-cidr=10.244.0.0/16
```

> Pod IP範圍(--pod-network-cidr)的相關含意將會在「Service & Networking」的章節中提到。


初始化後，會出現類似以下的訊息:

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```

依照訊息的提示，將管理員的 kubeconfig 設定好：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 關於 kubeconfig 將會在後面的章節中提到。這裡可以先簡單理解為「管理員對 cluster 的操作設定檔」。

初始化 Master Node 後，目前的 cluster 只存在一個 Node 而已，接下來我們要將 Worker Node 依序加入 cluster 中。

### STEP 6：加入 Worker Node

> 如果你安裝的是 single node cluster，請直接跳到「*STEP 7：安裝Pod network*」。

在 Master node 上初始化成功的輸出中，「最下方」有提示該如何加入 Worker Node，我們就直接複製該指令到所有 Worker Node 上執行即可：

> 以下操作僅於 worker node 上操作。

```bash
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

加入成功後，回到 master node 上，執行以下指令：

```bash
kubectl get nodes
# 會看到 Master Node 以及 Worker Node
```

雖然所有 Node 都已成功加入 cluster，但由於缺少 Pod network，所以 Node 的狀態會是 NotReady。

### STEP 7：安裝Pod network

為了讓 cluster 中的 Pod 可以彼此溝通，我們需要安裝 **CNI**(Container Network Interface) 來部署 Pod network，可參考[官方文件](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model)選則 CNI。

常見的 CNIs 例如 flannel、calico 等。這裡兩種安裝方式都會介紹：

> 這裡推薦安裝 calico，因為 calico 有支援後續章節會介紹的 NetworkPolicy。

**calico**

* 在 Master Node 上部署以下檔案：
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
```

* 由於我們剛才在 kubeadm init 時將「pod-network-cidr」設為 10.244.0.0/16，因此必須先下載 custom-resources 的檔案，修改後再進行安裝：

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml

vim custom-resources.yaml
```

* 修改如下：
```yaml
......
spec:
  # Configures Calico networking
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 # 修改這裡!
......
```

* 修改後部署 calico：
```bash
kubectl apply -f custom-resources.yaml
```

* 等待約兩分鐘，讓 calico 的相關 Pod 狀態變成 Running：

```bash
watch kubectl get pods -n calico-system
```

* 這時候再看看 Node 的狀態，直到變成 Ready，就代表 cluster 已經建置完成了：

```bash
kubectl get node -w
```

**flannel**

在 Master Node 上執行以下指令：

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

```bash
kubectl get nodes -w
# -w會持續監控node的狀態
```

等待一段時間後，當 Node 的狀態變成 Ready，就代表 cluster 已經建置完成了。

---
**提醒!**

> 如果你建置的是 single node cluster，記得繼續看下面的「Tips 4: Single node cluster」。

### 加入新的 Worker Node

如過在未來需要加入新的 Worker Node 到 Master 中，在 Master Node 上執行以下指令：
```bash
kubeadm token create --print-join-command
```

將上述指令的輸出複製並在新的 Worker Node 上執行，就可以加入 cluster 了。


### Tips 1：kubectl bash completion

Linux 的 bash shell 有一個很方便的功能，就是當你輸入指令時，按下「tab」鍵會自動補全。而 kubectl 也有支援這個功能，不過需要以下設定：

  * 下載 bash completion：
  ```bash
  sudo apt install bash-completion
  ```

  * 設定 kubectl 的 bash completion：
  ```bash
  echo 'source <(kubectl completion bash)' >>~/.bashrc
  source ~/.bashrc
  ```

  這樣就設定完成了，可以自行體驗一下，例如輸入「kubectl des」然後按下「tab」鍵，就會自動補全成「kubectl describe」。

  除此之外，為了更快速地下達指令，通常會將 kubectl 的 alias 設定成 `k`。而設定 alias 與相對應的 bash completion 設定方式如下：

  ```bash
  echo 'alias k=kubectl' >>~/.bashrc
  echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
  source ~/.bashrc
  ```

  同樣測試看看，如果輸入「k des」然後按下 tab 鍵，就會自動補全成「k describe」。

### Tips 2：在 Worker Node 使用 kubectl

假如你今天都沒有做任何設定，直接在 worker node 上執行 kubectl 指令，會發現它是無法執行的。

這是因為 kubectl 的設定檔是在 Master node 上，也就是 /etc/kubernetes/admin.conf，所以必須將 /etc/kubernetes/admin.conf 複製到 Worker Node 管理員的 $HOME/.kube/config (如同我們最初對 Master Node 所做的一樣)，才可以使用 kubectl 指令。

> 在 Worker Node 上執行以下操作：

```bash
mkdir -p $HOME/.kube
scp master:/etc/kubernetes/admin.conf ~/.kube/config
# 使用 scp 之前你需要先將相關的 ssh 設定好才能成功複製檔案
```

### Tips 3：初始化過程中的除錯

如果初始化 cluster 後出現各種問題，一定要多加利用 log 來除錯：

* 列出壞掉的容器:
```bash
sudo crictl ps -a
```

>  如果上述指令出現以下以下錯誤：

```text
ERRO[0000] unable to determine image API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory" 
```

> 原因是 socket 沒有設定好，使用以下指令修正：
```bash
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

* 查看容器的 log：
```bash
sudo crictl logs <container-id>
```

### Tips 4：Single node cluster

在預設中，k8s 為了讓 Master Node 與 Worker Node 各司其職，所以在 Master Node 上有「不能執行 Pod」的限制，這種限制稱為「taint」。

> 關於 taint 將會在後續的章節中介紹。

但是 single node cluster 的情況下，Master Node 需要同時肩負 worker node 的工作，所以需要移除 taint：

* 先找到 taint 的名稱：

```bash
kubectl describe node | grep -i taint
```
```text
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

* 移除taint：
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

這樣 Pod 未來就可以在 Master Node 上執行了，來部署一個簡單的 nginx pod 來測試一下：

```bash
kubectl run nginx --image=nginx
kubectl get pods -w
```

看到 Pod 的狀態變成 Running，就代表 single node cluster 建置完成了。

> 以上所有的 kubectl 操作都會在後續文章中陸續介紹。

### Tips 5：移除 cluster 中的 Worker Node

在 Master Node 上進行以下操作：

* 列出所有 Node：
```bash
kubectl get nodes

```

* 假設要移除的 Node 名稱是 worker1，我們將 worker1 的 Pod 先轉移到其他 Node 上，然後再移除 worker1：

```bash
kubectl drain worker1 --ignore-daemonsets 
```
```bash
kubectl delete node worker1
```
* 再次列出所有 Node，確認 worker1 已經被移除：
```bash
kubectl get nodes
```

如果要完全的清除 worker1 上 Kubernetes 的相關資料，在 worker1 上進行以下操作：

```bash
sudo kubeadm reset
```

* 清除所有相關的檔案:
```bash
sudo rm -rf /etc/kubernetes/
sudo rm -rf .kube/
sudo rm -rf /etc/cni/
```

* 最後，解除安裝和清除所有 kubernetes 的套件：
```bash
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube* 
sudo apt-get autoremove
```

## 今日小結

今天提供了兩種方式來建置練習環境。如果只是想練習一些基本操作，那 playground 應該就足夠了。但如果是練習多節點的操作，或想更全面的了解 cluster，那麼使用 kubeadm 來建置 cluster 對於初學者來說是一個不錯的選擇。


## 參考資料

* [Create a Kubernetes Cluster using Virtualbox — The Hard Way](https://medium.com/@mojabi.rafi/create-a-kubernetes-cluster-using-virtualbox-and-without-vagrant-90a14d791617)

* [Kubernetes Cluster Setup with Containerd](https://saurabhkharkate05.medium.com/kubernetes-cluster-setup-with-containerd-945214a0d02c)

* [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

* [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

* [kubectl completion](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_completion/)

* [Getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf)

* [Day 21：使用kubeadm建立集群](https://ithelp.ithome.com.tw/articles/10305268)

* [How to completely uninstall kubernetes](https://stackoverflow.com/questions/44698283/how-to-completely-uninstall-kubernetes)


* [Quickstart for Calico on Kubernetes](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

* [containerd cgroup setup](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

* [How to Gracefully Remove a Node from Kubernetes?](https://www.geeksforgeeks.org/gracefully-remove-a-node-from-kubernetes/)

