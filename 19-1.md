# Upgrade cluster

隨時留意官方版本的更新狀態，對於安全性提升和效能優化來說是個好習慣。`k8s`大約每三個月就會有些更新釋出，隨著版本的升級，我們勢必也需要
更新`cluster`。在[Day 03](03.md)中，我們使用`kubeadm`來建立`cluster`，因此今天就來介紹如何使用`kubeadm`來**升級**`cluster`。

## Upgrade strategy

粗略的說，升級`cluster`有這兩大階段:
  **step 1.** 升級`Master node` 
  **step 2.** 升級`Worker node`

> 先更新`master node`的原因在於，雖然更新期間`kube-apiserver`等重要元件無法使用，但跑在`worker node`上的`pod`還是可以正常運作。

至於`worker node`的更新，有以下幾種策略可以選擇:
  1. 一次性關閉所有`worker node`，然後一次更新完畢
  2. 一次`drain`一個`worker node`，依序更新直到全部更新完畢
  3. 準備一個新的`worker node`，將`pod`移過去，然後淘汰舊的`worker node`，如此循環直到全部更新完畢

在雲端的環境中，第三種策略是相當常見的，因為新增一台新的機器只需要動動手指就搞定了。那如果是非雲端的環境，第一種方式雖然簡單粗暴，但是會造成服務的中斷，因此第二種方式是比較常見的，底下也會以第二種方式來示範。

## 升級cluster範例

**升級目標**: 將`cluster`升級到`v1.29.0`

### Step 1. 升級`Master node`

> 在**master node**上進行以下操作:

確認目前`cluster`的版本:
```bash
$ kubectl get node
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   6d17h   v1.28.4
node01         Ready    <none>          6d17h   v1.28.4
node02         Ready    <none>          6d17h   v1.28.4
```
`drain`掉`master node`:
```bash
kubectl drain controlplane --ignore-daemonsets
```

接著查看可用的新版本:
```bash
sudo apt update
sudo apt-cache madison kubeadm
   kubeadm | 1.28.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
   kubeadm | 1.28.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.28/deb  Packages
```

但是並沒有找到'v1.29.0'的版本(*如果有的話就可以跳過下面修改/etc/apt/sources.list.d/kubernetes.list/的步驟*)，所以先來確認`apt`針對`k8s`的`repository`版本:

**情況1**
```bash
 $ cat /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /
```

或者你可能看到這樣:

**情況2**
```bash
$ cat /etc/apt/sources.list.d/kubernetes.list

deb [signed-by=/etc/apt/keyrings/kubernetes-1-27-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /
deb [signed-by=/etc/apt/keyrings/kubernetes-1-28-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /
```

> 如果是看到**情況1**，請繼續往下操作，如果是**情況2**，請參考[這裡](19-2-apt-source.md)

原來版本設定是`v1.28`，所以我們需要先更新`repository`的版本:
```bash
$ sudo vim /etc/apt/sources.list.d/kubernetes.list
# 修改如下
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

再次確認可用的新版本:
```bash
sudo apt update
sudo apt-cache madison kubeadm
  kubeadm | 1.29.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
  kubeadm | 1.29.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
  kubeadm | 1.29.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.29/deb  Packages
```

這裡選擇用`1.29.0-1.1`版本來升級。先升級`kubeadm`:
```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.29.0-1.1*' && \
sudo apt-mark hold kubeadm
```

更新後確認`kubeadm`的版本:
```bash
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"29", GitVersion:"v1.29.0", GitCommit:"3f7a50f38688eb332e2a1b013678c6435d539ae6", GitTreeState:"clean", BuildDate:"2023-12-13T08:50:10Z", GoVersion:"go1.21.5", Compiler:"gc", Platform:"linux/amd64"}
```

可以看到`kubeadm`的版本已經更新到`v1.29.0`。接著確認升級的可行性:
```bash
$ kubeadm upgrade plan
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     2 x v1.28.4   v1.29.2

Upgrade to the latest stable version:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.28.4   v1.29.2
kube-controller-manager   v1.28.4   v1.29.2
kube-scheduler            v1.28.4   v1.29.2
kube-proxy                v1.28.4   v1.29.2
CoreDNS                   v1.10.1   v1.11.1
etcd                      3.5.9-0   3.5.10-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.29.2

Note: Before you can perform this upgrade, you have to update kubeadm to v1.29.2.
```

出現以上畫面就可以開始升級`master node`了。雖然提示叫我們更新到`v1.29.2`，但我們還是依計畫選擇升級到`1.29.0`:
> 當然，你也可以選擇依照提示更新，不過就需要再次更新`kubeadm`到`v1.29.2`。

```bash
sudo kubeadm upgrade apply v1.29.0
...
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
# 看到以上提示代表`master node`已經升級完畢(包括重要元件)
```

接著更新`kubelet`和`kubectl`:
```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.29.0-1.1*' kubectl='1.29.0-1.1*' && \
sudo apt-mark hold kubelet kubectl
```

接著重啟`kubelet`:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

這樣`master node`就升級完畢了。確認一下:
```bash
$ kubectl get node
NAME           STATUS                     ROLES           AGE     VERSION
controlplane   Ready,SchedulingDisabled   control-plane   6d18h   v1.29.0
node01         Ready                      <none>          6d18h   v1.28.4
```

接著讓`master node`恢復成`Schedulable`:
```bash
kubectl uncordon controlplane
```

準備更新`worker node`，先`drain`掉`node01`:
```bash
kubectl drain node01 --ignore-daemonsets
```

### Step 2. 升級`Worker node`

> 在**worker node**上進行以下操作:

切換到`worker node`上:
```bash
$ ssh node01
```
> 先check一下/etc/apt/sources.list.d/kubernetes.list的內容，如果是v1.28的版本，同樣以上面`master node`的步驟修改即可

先更新`kubeadm`:
```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.29.0-1.1*' && \
sudo apt-mark hold kubeadm
```
升級`worker node`:
```bash
sudo kubeadm upgrade node
```

更新`kubelet`和`kubectl`(其實步驟和`master node`很像):
```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.29.0-1.1*' kubectl='1.29.0-1.1*' && \
sudo apt-mark hold kubelet kubectl
```

重啟`kubelet`:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

這樣整個`cluster`就升級完畢了。`exit`回到`master node`確認一下:
```bash
$ kubectl get node
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   6d18h   v1.29.0
node01         Ready    <none>          6d18h   v1.29.0
```

最後將`node01`恢復成`Schedulable`:
```bash
kubectl uncordon node01
```

