# *Day03 Basic Concept* : 建置練習環境

在接下來的章節中，將會有許多範例或練習需要對`cluster`進行操作，所以我們需要事先準備一個練習的環境，以下提供了兩種方法進行建置:

## 方法一 : playground

這是最簡單輕鬆的方式，直接使用網路上的`playground`進行練習，不需要額外的環境建置。以下提供了兩個網站，登入後就能得到一個*暫時性*的`cluster`，可以進行練習:
  1. [killercoda](https://killercoda.com/)
  2. [Play with Kubernetes](https://labs.play-with-k8s.com/)

## 方法二 : kubeadmin

不過使用`playground`的方式，練習的結果是暫時性的。所以如果想要建立一個較為完整的`cluster`，可以使用`kubeadmin`進行建置。`kubeadmin`是一個專門用來部署`Kubernetes`的工具，能夠快速的建立一個`cluster`，並且可以直接在本地端進行操作。以下將以virtualbox為例，介紹如何使用`kubeadmin`進行建置。

### Step 1 : 準備環境

首先需要安裝`virtualbox`，並建立至少兩台的VM，一台作為`master node`，其他的作為`worker node`。需注意的是，每台VM只少需要:
  * 2GB RAM
  * 2 CPU

在網路設定方面，每台虛擬機準備兩張網路卡:
  * 一張選用`NAT`方便連上網際網路
  * 一張選用橋接介面卡，方便進行`cluster`內部的VM溝通。記得這張不要用`DHCP`，需要自行手動設定IP。例如下表:
  
  VM | IP
  ---|---
  master | 192.168.132.1
  worker1 | 192.168.132.2
  worker2 | 192.168.132.3

至於安裝VM與設定IP的方式就不再此贅述，可以參考以下文章:
  * [在virtualbox上安裝Ubuntu](https://karenkaods.medium.com/%E4%B8%89%E6%AD%A5%E9%A9%9F%E5%9C%A8-windows-%E9%9B%BB%E8%85%A6%E4%B8%8A%E5%AE%89%E8%A3%9D-vitrualbox-%E5%95%9F%E5%8B%95-ubuntu-%E8%99%9B%E6%93%AC%E6%A9%9F-f45619d3c088)
  * [設定Ubuntu IP](https://sam.liho.tw/2022/09/29/ubuntu-22-04-%E6%8C%87%E4%BB%A4-cli-%E8%A8%AD%E5%AE%9A%E7%B6%B2%E8%B7%AF%E7%AD%86%E8%A8%98/)

### Step 2 : 安裝Docker
**每台**VM上都需要安裝container runtime，這裡以`Docker`為例:
```text
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
```
安裝Docker後需要更改cgroup driver，因為Kubernetes預設使用`systemd`，而`Docker`預設使用`cgroupfs`。所以需要更改`Docker`的cgroup driver:
```text
sudo cat <<EOF | sudo tee /etc/docker/daemon.json
{ "exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts":
{ "max-size": "100m" },
"storage-driver": "overlay2"
}
EOF
```
接著重新啟動`Docker`:
```text
sudo systemctl restart docker
```

這樣cgroup driver就改成`systemd`了:
```text
docker info | grep -i cgroup
# cgroup Driver: systemd
```

### Step 3 : 安裝必要組件

在每台VM上，需要以下三個組件:
  * `kubelet`: [上一個章節](02.md)有提到，它是「小船的船長」
  * `kubeadm`: 用來部署`cluster`的工具
  * `kubectl`: 用來與`cluster`進行溝通的cli工具，讓你能透過下指令的方式操作`cluster`

安裝以上三個組件的方式如下:
* 首先，把kubernetes的repo加入到apt的source list中
```text
sudo apt update && sudo apt install -y curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

* 查看目前可用的`kubeadm`版本
```text
apt list -a kubeadm
# 這裡會列出許多版本，以下範例選用1.28.0-00
```

* 安裝`kubelet`、`kubeadm`、`kubectl`
```text
sudo apt-get update
sudo apt-get install -y kubelet=1.28.0-00 kubeadm=1.28.0-00 kubectl=1.28.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

* 檢查`kubeadm`是否安裝成功 
```text
kubeadm version
```
### Step4 : 關閉swap並啟用ip_forward
在預設上，如果swap沒有被關閉，可能會導致`kubelet`無法正常運作。所以需要先關閉swap:
```text
sudo swapoff -a # 暫時關閉
vim /etc/fstab # 若想要永久關閉，可以將swap的那一行註解掉
```

啟用ip_forward:
```text
sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

### Step 5 : 初始化master node

初始化時，記得指定apiserver的IP:
> 以下操作僅於master node 上操作。
```text
sudo kubeadm init --apiserver-advertise-address <master node IP> --control-plane-endpoint <master node IP> \ 
--pod-network-cidr=10.244.0.0/16
```

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

依照訊息的提示，將管理員的kubeconfig設定好:
```text
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> 關於`kubeconfig`的介紹將會在後面的章節中提到。這裡可以先簡單理解為「管理員對`cluster`的操作設定檔」。

### Step 6 : 加入worker node
在master node上初始化成功的輸出中，最下方有提示該如何加入worker node，我們就直接依照指示操作即可:
> 以下操作僅於worker node上操作。
```text
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

加入成功後，回到master node上，執行以下指令:
```text
kubectl get nodes
# 會看到master node以及worker node
```
雖然已經加入成功，但由於缺少`Pod network`，所以`node`的狀態會是`NotReady`。

### Step 7 : 安裝Pod network

`Pod`之間必須透過`CNI`，也就是`Container Network Interface`進行溝通。所以需要安裝`CNI`，這裡以常見的`flannel`為例:
```text
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
回到master node上，執行以下指令:
```text
kubectl get nodes -w
# -w會持續監控node的狀態
```

等待一段時間後，當`node`的狀態變成`Ready`，就代表`cluster`已經建置完成了。

### 加入新的worker node

如過在未來需要加入新的`worker node`，需要在master node上執行以下指令:
```text
kubeadm token create --print-join-command
```

將上述指令的輸出複製並在新的`worker node`上執行，就可以加入`cluster`了。


### 補充1: kubectl bash completion

Linux的bash shell有一個很方便的功能，就是當你輸入指令時，按下`tab`鍵會自動補全。而`kubectl`也有這個功能，不過需要以下設定:

  * 下載bash completion:
  ```text
  sudo apt install bash-completion
  ```

  * 設定`kubectl`的bash completion:
  ```text
  echo 'source <(kubectl completion bash)' >>~/.bashrc
  source ~/.bashrc
  ```

  這樣就設定完成了，可以自行體驗一下，例如輸入"kubectl des"然後按下`tab`鍵，就會自動補全成"kubectl describe"。

  > 除此之外，為了更快速地下達指令，通常會將`kubectl`的alias設定成`k`。而設定alias的方式與相對應的bash completion設定方式如下:

  ```text
  echo 'alias k=kubectl' >>~/.bashrc
  echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
  source ~/.bashrc
  ```

  同樣測試看看，如果輸入"k des"然後按下`tab`鍵，就會自動補全成"k describe"。

### 補充2: 在cluster的worker node使用kubectl

假如你今天都沒有做任何設定，直接在`worker node`上執行`kubectl`指令，會發現它是無法執行的。這是因為`kubectl`的設定檔是在`master node`上，也就是`/etc/kubernetes/admin.conf`，所以必須將`/etc/kubernetes/admin.conf`複製到`worker node`上的`$HOME/.kube/config`(如同我們最初對master node所做的一樣)，才可以使用`kubectl`指令。在`worker node`上執行以下操作:

```text
mkdir -p $HOME/.kube
scp master:/etc/kubernetes/admin.conf ~/.kube/config
```

## 今日小節
今天提供了兩種方式來建置練習環境。如果只是想練習一些基本操作，那playground應該就足夠了。但如果是練習多節點的操作，或想更全面的了解`cluster`，那麼使用`kubeadm`來建置`cluster`對於初學者來說是一個不錯的選擇。(補充: 之所以說初學者，是因為kubeadm還是相對簡單，如果有興趣，可以上網搜尋"Kubernetes The Hard Way ")

## 參考資料
* [Create a Kubernetes Cluster using Virtualbox — The Hard Way](https://medium.com/@mojabi.rafi/create-a-kubernetes-cluster-using-virtualbox-and-without-vagrant-90a14d791617)

* [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

* [Day 21：使用kubeadm建立集群](https://ithelp.ithome.com.tw/articles/10305268)
