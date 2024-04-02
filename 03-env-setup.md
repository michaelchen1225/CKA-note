# *Day03 Basic Concept* : 建置練習環境

在接下來的章節中，將會有許多範例或練習需要對`cluster`進行操作，所以我們需要事先準備一個練習的環境，以下提供了兩種方法進行建置:

## 方法一 : playground

這是最簡單輕鬆的方式，直接使用網路上的`playground`進行練習，不需要額外的環境建置。以下提供了兩個網站，登入後就能得到一個*暫時性*的`cluster`，可以進行練習:
  1. [killercoda](https://killercoda.com/)
  2. [Play with Kubernetes](https://labs.play-with-k8s.com/)

## 方法二 : kubeadm

不過使用`playground`的方式，練習的結果是暫時性的。所以如果想要建立一個較為完整的`cluster`，可以使用`kubeadm`進行建置。`kubeadm`是一個專門用來部署`Kubernetes`的工具，能夠快速的建立一個`cluster`，並且可以直接在本地端進行操作。以下將以virtualbox為例，介紹如何使用`kubeadm`進行建置。

使用`kubeadm`進行建置`cluster`的步驟如下:

  1. 準備環境 
  2. 安裝container runtime
  3. 安裝必要組件: kubelet、kubeadm、kubectl
  4. 關閉swap並啟用ip_forward
  5. 初始化master node
  6. 加入worker node
  7. 安裝Pod network

### Step 1 : 準備環境

首先需要安裝`virtualbox`，並建立至少兩台的VM，一台作為`master node`，其他的作為`worker node`。需注意的是，每台VM只少需要:
  * 2GB RAM
  * 2 CPU

在網路設定方面，每台虛擬機準備兩張網路卡:
  * 一張選用`NAT`
  * 一張選用橋接介面卡，方便進行`cluster`內部的VM溝通。記得這張不要用`DHCP`，需要自行手動設定IP。例如下表:
  
  VM | IP
  ---|---
  master | 192.168.132.1
  worker1 | 192.168.132.2
  worker2 | 192.168.132.3

至於安裝VM與設定IP的方式就不再此贅述，可以參考以下文章:
  * [在virtualbox上安裝Ubuntu](https://karenkaods.medium.com/%E4%B8%89%E6%AD%A5%E9%A9%9F%E5%9C%A8-windows-%E9%9B%BB%E8%85%A6%E4%B8%8A%E5%AE%89%E8%A3%9D-vitrualbox-%E5%95%9F%E5%8B%95-ubuntu-%E8%99%9B%E6%93%AC%E6%A9%9F-f45619d3c088)
  
  * [設定Ubuntu IP](https://sam.liho.tw/2022/09/29/ubuntu-22-04-%E6%8C%87%E4%BB%A4-cli-%E8%A8%AD%E5%AE%9A%E7%B6%B2%E8%B7%AF%E7%AD%86%E8%A8%98/)

**補充**

其實只用一台VM即可建立`cluster`，這樣的方式稱為`single node cluster`。

如果想要建立single node cluster，同樣按照下面的步驟進行即可，不過要注意做完「Step 5」後，不須操作「*Step 6 : 加入worker node*」，直接跳到「*Step 7 : 安裝Pod network*」即可。


### Step 2 : 安裝container runtime

**每台**VM上都需要安裝container runtime，這裡以`containerd`為例:

> 不同版本的安裝步驟可以參考[官方文件](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf)，這裡以`Ubuntu`為例。

設定好`apt repo`的GPG key:
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

加入`containerd`的repo:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

安裝`containerd`:
```bash
sudo apt-get install containerd.io
sudo systemctl enable containerd
```
安裝後檢查一下是否有正常運作:
```bash
systemctl status containerd
```

> containerd和docker都是container runtime， 兩者的差異可以參考[這裡](https://cloud.tencent.com/document/product/457/35747)

安裝後，需要將cgroup-driver設定為`k8s cluster`所需的`systemd`:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

編輯`/etc/containerd/config.toml`，找到「[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]」，將`SystemdCgroup`設定為`true`:
```yaml
...
.....(省略)....
...
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

...
.....(省略)....
```

最後重新啟動`containerd`:
```bash
sudo systemctl restart containerd
```

最終檢查一下是否有將「SystemdCgroup=ture」設定成功:
```bash
containerd config dump | grep SystemdCgroup
# SystemdCgroup = true
```
**提醒**

請確保上述步驟有設定成功，否則cluster建立後重要元件會不斷重啟而無法使用!

### Step 3 : 安裝必要組件: kubelet、kubeadm、kubectl

在每台VM上，需要以下三個組件:
  * `kubelet`: [上一個章節](02.md)有提到，它是「小船的船長」
  * `kubeadm`: 用來部署`cluster`的工具
  * `kubectl`: 用來與`cluster`進行溝通的cli工具，讓你能透過下指令的方式操作`cluster`

安裝以上三個組件的方式如下:

* 看這裡，底下可能要改: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

* 首先，把kubernetes的repo加入到apt的source list中
```bash
sudo apt-get update

sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# 如果顯示錯誤"etc/apt/keyrings does not exist"。請先執行"sudo mkdir -p -m 755 /etc/apt/keyrings，再重新curl

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
> 如果是其他版本的Linux，可參考[官方文件](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

* 查看目前可用的`kubeadm`版本
```bash
sudo apt update
sudo apt-cache madison kubeadm
# 這裡會列出許多版本，以下範例選用1.29.1-1.1
```

> 當然也可以自行選擇其他版本，不過要記得下面的指令不要照抄

* 安裝`kubelet`、`kubeadm`、`kubectl`

```bash
sudo apt-get update
sudo apt-get install -y kubelet=1.29.1-1.1 kubeadm=1.29.1-1.1 kubectl=1.29.1-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

* 檢查`kubeadm`是否安裝成功 
```bash
kubeadm version
```
### Step4 : 關閉swap並啟用ip_forward
在預設上，如果swap沒有被關閉，可能會導致`kubelet`無法正常運作。所以需要先關閉swap:
```bash
sudo swapoff -a # 暫時關閉
sudo vim /etc/fstab # 若想要永久關閉，可以將swap的那一行註解掉
```

載入必要模組，啟用ip_forward:
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
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

### Step 5 : 初始化master node

初始化時，記得指定apiserver的IP:
> 以下操作僅於master node 上操作。
```bash
sudo kubeadm init --apiserver-advertise-address <master node IP> --control-plane-endpoint <master node IP> --pod-network-cidr=10.244.0.0/16
```

> pod IP範圍(`--pod-network-cidr`)的相關含意將會在[Day 28](28-network.md)中提到

初始化後，會出現類似以下的訊息:
```bash
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
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> 關於`kubeconfig`的介紹將會在後面的章節中提到。這裡可以先簡單理解為「管理員對`cluster`的操作設定檔」。

### Step 6 : 加入worker node

> 如果你安裝的是`single node cluster`，請直接跳到「*Step 7 : 安裝Pod network*」。

在master node上初始化成功的輸出中，最下方有提示該如何加入worker node，我們就直接依照指示操作即可:
> 以下操作僅於worker node上操作。
```bash
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

加入成功後，回到master node上，執行以下指令:
```bash
kubectl get nodes
# 會看到master node以及worker node
```
雖然已經加入成功，但由於缺少`Pod network`，所以`node`的狀態會是`NotReady`。

### Step 7 : 安裝Pod network

你可以選用常見的CNIs，例如`flannel`、`calico`等。這裡兩種安裝方式都會介紹:

**flannel**

回到master node上，執行以下指令:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

```bash
kubectl get nodes -w
# -w會持續監控node的狀態
```

等待一段時間後，當`node`的狀態變成`Ready`，就代表`cluster`已經建置完成了。

**calico**

* 直接部署以下檔案:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
```

* 由於我們剛才在kubeadm init時將`pod-network-cidr`設為10.244.0.0/16，因此必須先下載下面的檔案，修改後再進行安裝:

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml

vim custom-resources.yaml
```

* 修改如下:
```yaml
...(省略)...
...
spec:
  # Configures Calico networking
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
...
...(省略)...
```

* 修改後部署calico:
```bash
kubectl apply -f custom-resources.yaml
```

* 等待一下讓calico的相關pod狀態變成`Running`:

```bash
watch kubectl get pods -n calico-system
```

* 這時候再看看node的狀態，直到變成`Ready`，就代表`cluster`已經建置完成了:

```bash
kubectl get node -w
```

**提醒!**

如果你建置的是`single node cluster`，請繼續看「補充4」。

### 加入新的worker node

如過在未來需要加入新的`worker node`，需要在master node上執行以下指令:
```bash
kubeadm token create --print-join-command
```

將上述指令的輸出複製並在新的`worker node`上執行，就可以加入`cluster`了。


### 補充1: kubectl bash completion

Linux的bash shell有一個很方便的功能，就是當你輸入指令時，按下`tab`鍵會自動補全。而`kubectl`也有這個功能，不過需要以下設定:

  * 下載bash completion:
  ```bash
  sudo apt install bash-completion
  ```

  * 設定`kubectl`的bash completion:
  ```bash
  echo 'source <(kubectl completion bash)' >>~/.bashrc
  source ~/.bashrc
  ```

  這樣就設定完成了，可以自行體驗一下，例如輸入"kubectl des"然後按下`tab`鍵，就會自動補全成"kubectl describe"。

  > 除此之外，為了更快速地下達指令，通常會將`kubectl`的alias設定成`k`。而設定alias的方式與相對應的bash completion設定方式如下:

  ```bash
  echo 'alias k=kubectl' >>~/.bashrc
  echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
  source ~/.bashrc
  ```

  同樣測試看看，如果輸入"k des"然後按下`tab`鍵，就會自動補全成"k describe"。

### 補充2: 在cluster的worker node使用kubectl

假如你今天都沒有做任何設定，直接在`worker node`上執行`kubectl`指令，會發現它是無法執行的。這是因為`kubectl`的設定檔是在`master node`上，也就是`/etc/kubernetes/admin.conf`，所以必須將`/etc/kubernetes/admin.conf`複製到`worker node`上的`$HOME/.kube/config`(如同我們最初對master node所做的一樣)，才可以使用`kubectl`指令。在`worker node`上執行以下操作:

```bash
mkdir -p $HOME/.kube
scp master:/etc/kubernetes/admin.conf ~/.kube/config
```

### 補充3: 除錯

如果初始化cluster後出現各種問題，一定要多加利用log來除錯:

* 列出壞掉的容器:
```bash
sudo crictl ps -a
```

>  如果上述指令出現以下以下錯誤:
```text
WARN[0000] runtime connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
ERRO[0000] unable to determine runtime API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory" 
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
ERRO[0000] unable to determine image API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory" 
```

> 請用以下指令修正:
```bash
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

* 查看容器的log:
```bash
sudo crictl logs <container-id>
```

### 補充4: Single node cluster

在預設中，k8s為了讓master node與worker node各司其職，所以在master node上有「不能運行pod」的限制，這種限制稱為「taint」。

> 關於taint的介紹將會在後面的章節中提到。

但是single node cluster的情況下，master node也是worker node，所以需要移除taint:

* 先找到taint的名稱:

```bash
kubectl describe node | grep -i taint
```

```bash
# 輸出如下:
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

* 移除taint:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

這樣就完成了，來部署一個簡單的nginx pod來測試一下:

```bash
kubectl run nginx --image=nginx
kubectl get pods -w
```

看到pod的狀態變成`Running`，就代表single node cluster建置完成了。


### 補充5: 心灰意冷，想重新來過

如果你覺得完全搞砸了、沒救了想直接砍掉重來，可以使用`kubeadm reset`:
```bash
sudo kubeadm reset
```
接著清除所有相關的檔案:
```bash
sudo rm -rf /etc/kubernetes/
sudo rm -rf .kube/
sudo rm -rf /etc/cni/
```

最後，解除安裝和清除所有kubernetes的套件:
```bash
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube* 
sudo apt-get autoremove
```

> 不過建議還是先試著用除錯的方式來解決問題，或是查找相關資料，不要放棄!


## 今日小節
今天提供了兩種方式來建置練習環境。如果只是想練習一些基本操作，那playground應該就足夠了。但如果是練習多節點的操作，或想更全面的了解`cluster`，那麼使用`kubeadm`來建置`cluster`對於初學者來說是一個不錯的選擇。(補充: 之所以說初學者，是因為kubeadm還是相對簡單，如果有興趣，可以上網搜尋"Kubernetes The Hard Way ")

## 參考資料
* [Create a Kubernetes Cluster using Virtualbox — The Hard Way](https://medium.com/@mojabi.rafi/create-a-kubernetes-cluster-using-virtualbox-and-without-vagrant-90a14d791617)

* [Kubernetes Cluster Setup with Containerd](https://saurabhkharkate05.medium.com/kubernetes-cluster-setup-with-containerd-945214a0d02c)

* [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

* [Day 21：使用kubeadm建立集群](https://ithelp.ithome.com.tw/articles/10305268)

* [How to completely uninstall kubernetes](https://stackoverflow.com/questions/44698283/how-to-completely-uninstall-kubernetes)


* [Quickstart for Calico on Kubernetes](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

* [containerd cgroup setup](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

