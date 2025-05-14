# Day 14 -【Storage】： PV、PVC & StorageClass

### 今日目標

* 了解 PV、PVC、StorageClass 的概念與關係

* 實際使用 PV、PVC、StorageClass
  * 使用 nfs 作為 storageClass 的 provisioner (optional)

### 什麼是 PV、PVC、StorageClass

上個章節介紹了 volume，但如果我們只是單純的使用 volume 會有以下幾個缺點：

  * Volume 的生命週期和 Pod 是一樣的，當 Pod 被刪除時，volume 也會被刪除。
  
  * 假如定義了 hostPath，當 Pod 轉移到另一個 Node 上時，我們無法保證新的 Node 上也有同樣的「hostPath」，可能造成 Pod 讀不到資料。

  * 每個 Pod 都要定義自己的 volume，當 Pod 數量增加時，定義這些不能重複使用的 volume 會變得很麻煩。


如果能先建立一個獨立於 Pod 之外能持續存活的「volume」，等 Pod 有需要時再依需求分配，不需要的時候就釋出，這樣就方便許多了。而這樣的「volume」就是 **Persistent Volume** (PV)，而 Pod 對 PV 的需求就透過 **Persistent Volume Claim** (PVC) 來提出。

> 建立 PVC 來要求儲存空間，系統會依照 PVC 的要求給予相對應的 PV，最終再將 PVC 與 Pod 綁定，這樣 Pod 就有儲存空間可以使用了。


而 PV 的配置方式可以分為以下兩種模式：

 * **Static**：由管理者自行定義並建置，也就是使用 yaml 建立 PV。

 * **Dynamic**: 由 StorageClass 自動建置。

一個 cluster 中能存在多種 storageClass，我們能依照不同需求指定 PV、PVC所屬的 storageClass，這樣就能有效的分類不同的儲存資源，就像將 Pod 歸類到不同 namespace 一樣。

而 StorageClass 可以讓 PV 的配置更加方便與彈性，舉例來說，我們建立了一個屬於 storageClass「A」的 PVC 時，管理者不用像以前一樣手動建立 PV，因為 storageClass 「A」看到新的 PVC 後就會自動建立相對應的 PV。

不過，PV 與 PVC 皆無法使用 `kubectl create` 直接建立，所以底下我們得先了解兩者的 yaml 寫法。 

### PV 與 PVC 的 yaml 格式

我們先來看看 PV 的 yaml 格式:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>
spec:
  capacity:
    storage: <storage-size>
  accessModes:
    - <access-mode>
  persistentVolumeReclaimPolicy: <reclaim-policy>
  <pv-type>:
    <pv-type-configurations>
```

 * **capacity**：定義 PV 的大小，例如: 1Gi、500Mi 等。

 * **accessModes**：定義 PV 的存取模式，有以下三種：
 
   * ReadWriteOnce (RWO)：只能被**單一** Node 掛載為讀寫模式

   * ReadOnlyMany (ROX)：可以被**多個** Node 掛載為唯讀模式

   * ReadWriteMany (RWX)：可以被**多個** Node 掛載為讀寫模式


 * **persistentVolumeReclaimPolicy**：定義當 PVC 被刪除時的行為，有以下兩種：
 
   * Retain：保留 PV 裡的資料，除非被管理員手動刪除，否則不能再被其他 PVC 使用 (預設值)。

   * Delete: 直接刪除 PV 以及相關的儲存資源。

 * **pv-type**: 定義 PV 的類型，例如: `nfs`、`hostPath`、`local` 等。其他的 pv-type 可參考[官網](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)

 * **pv-type-configurations**: 根據不同的 pv-type 而有不同的配置，例如：nfs 需要定義 server、path 等

> 以上只是最基本的 PV 配置，其他的可以參考[官網](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)

我們再來看看 PVC 的 yaml 格式：

 ```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
spec:
  accessModes:
    - <access-mode>
  resources:
    requests:
      storage: <request-size>
```

* **accessModes**：同 PV 的 accessModes

* **resources**：定義所需的大小，例如: 1Gi、500Mi 等

另外，也可以使用 selector 或 volumeName 來指定 PV：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
spec:
  accessModes:
    - <access-mode>
  resources:
    requests:
      storage: <request-size>
  selector: # 使用selector來指定PV
    matchLabels:
      <key>: <value>
```
```yaml
......
spec:
  volumeName: <pv-name> # 使用volumeName來指定PV
  accessModes:
    - <access-mode>
...
```

當建立 PVC 後，K8s 會自動尋找符合條件的 PV，如果符合條件的 PV 不存在，則 PVC 會一直處於 Pending 狀態，直到符合條件的PV被建立。

所謂的「符合條件」例如：
  
  * accessModes 是否符合

  * PV 的 capacity 是否大於或等於 PVC 的 request

  * selector 是否符合

  * storageClass 是否符合

---
> **注意**

* 假設 PV 的 capacity 是 1Gi，PVC 的 request 是 500Mi，若該 PV 與 PVC 綁定了，實際上 PVC　會拿到 1Gi 的空間，而不是 500Mi。
* PV 與 PVC 是一對一關係，PV 一旦和某個 PVC 綁定後，綁定期間就不能再被其他 PVC 使用。
***

### StorageClass 的 yaml 格式

storage class 的 yaml 格式如下：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: <storage-class-name>
provisioner: <provisioner>
reclaimPolicy: <reclaim-policy>
allowVolumeExpansion: <true or false>
volumeBindingMode: <volume-binding-mode>
```

* **provisioner**：storageClass 會建立 PV，就需要一個 PV 的「提供者」。可用的 provisioner 清單請見[官網](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)。

* **reclaimPolicy**：同 PV 的 ReclaimPolicy，預設為 Delete

* **allowVolumeExpansion**：是否允許 PVC 要求更多的 size，**true**為允許。

* **volumeBindingMode**： 定義 PV 的綁定模式，有以下兩種：

  * Immediate：當 PVC 被建立時，storageClass 會立即建立 PV

  * WaitForFirstConsumer：當 PVC 被建立時，storageClass 不會立即建立 PV，而是等到有 Pod 使用 PVC 時才建立 PV (底下有關 local pv 的例子會使用到)

> 以上是最基本的 storageClass 配置，其他設定可以參考[官網](hhttps://kubernetes.io/docs/concepts/storage/storage-classes/)

### 實作：hostPath Persistent Volume

> **目標**：先準備一個檔案在 PV 中，透過 PVC 將該檔案掛載到 Pod 。

* 在 node01 建立測試用的檔案(如果是 single-node cluster，則在 master node 上建立)：
    
```bash
ssh node01
echo "testing pv and pvc" > /tmp/test.txt
```

* 接著 ssh 回到 Master Node，建立一個 PV：

```yaml
# hostpath-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  storageClassName: just-test
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp
```
```bash
kubectl apply -f hostpath-pv.yaml
```

---
> **Tips**：編造 storageClassName

這個「just-test」 storageClassName 是我們隨便取的，不用**真的**建立一個 storage class。

因為如果不指定 storageClassName，則會使用系統的 default storage class，這樣後面 PVC 與 Pod 建立後，用的就不會是我們自訂的 PV，而是預設 storage class 給的 PV

所以使用「編造 storageClassName」的技巧，就能繞開預設的 storage class，讓我們自己定義的 PV 被使用。
***

* 建立後檢查 PV 的狀態：
```bash
kubectl get pv test-pv
```
```text
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
test-pv   500Mi      RWO            Retain           Available           just-test      <unset>                          3s

```
> STATUS為「Available」，表示 PV 還沒被使用，可以被 PVC 請求

* 再建立一個 PVC：

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  storageClassName: just-test
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
```bash
kubectl apply -f test-pvc.yaml
```

* 檢查 PVC 的狀態：
```bash
kubectl get pvc test-pvc
```
```text
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound    test-pv   500Mi      RWO            just-test      <unset>                 3s
```
> STATUS 為「Bound」，表示 PVC 已成功綁定到 PV 上。

* 建立一個Pod，掛載 PVC：

```yaml
# hostpath-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-nginx
spec:
  nodeName: node01  # 確保 Pod 能存取到 hostPath
  volumes:
    - name: pv-volume
      persistentVolumeClaim:
        claimName: test-pvc
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-volume
```
```bash
kubectl apply -f hostpath-nginx.yaml
```

建立後查看一下是否掛載成功：
```bash
kubectl exec -it hostpath-nginx -- cat /usr/share/nginx/html/test.txt
```
如果掛載有成功，輸出如下：
```txt
testing pv and pvc
```

### hostPath 的缺點

我們知道 Pod 會被 schduler 分配到不同的 Node 上，假設今天有一個 Pod 使用了 hostPath 的 PV，但是卻被分配到**另一個** Node 上，而如果該 Node 上沒有相同的 hostPath，Pod 就會讀不到資料。

當然，我們可以使用 nodeName 或 nodeSelector 來指定 Pod 要跑在哪個 Node 上，但當今天面對一個複雜的 cluster、有一堆 Node 時，這樣的方式就會很麻煩。(註：有關 nodeName、nodeSelector 的使用方式會在後續章節介紹)

所以，hostPath 的缺點就是**不具備跨 Node 的能力**，所以常見的使用環境為「single-node cluster」。

為了解決這個問題，我們可以使用另一種 PV 類型：**local**。

### 實作：local Persistent Volume

local 類型的 PV 會掛載「特定 Node」上的儲存空間，例如硬碟、分割槽、目錄，聽起來似乎與 hostPath 很像，但差別就在於「特定的 Node」，也就是說，當 scheduler 分配 Pod 時，它無法保證將 Pod 分配到有 hostPath 的 Node 上，但是能夠保證將 Pod 分配到有 local PV 的 Node 上。

為了做到將 PV 建立在正確的 Node 上，我們必須在 PV 的 yaml 中加入 nodeAffinity (後續章節會再介紹)。我們來看一個例子：

> 我們想確保 Pod 一定能存取到 node01 上的 /test 目錄，以下依照需求來建立 local PV：

* 在 node01 上建立 /test 目錄，並在裡面創建一個檔案：

```bash
ssh node01
sudo mkdir -p /test
echo "testing local pv" > /test/test.txt
```

* 返回 Master Node 上，建立 storage class：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
```bash
kubectl apply -f storage-class.yaml
```

***
> **Tips：** 利用 storageClass 延遲 PV 與 PVC 的綁定時間

你或許會好奇為什麼要建立 storageClass？

在沒有 storageClass 的情況下，假設有一個 local 類型的 PV-1 在 node01 上，當 PVC-1 建立時，系統可能會自行將 PV-1 與 PVC-1 綁定。 

接著，有一個 nginx-pod **並非**要使用 PV-1 的資料，我們指定要 nginx-pod 被分配到 node02，但指定了該 Pod 使用 PVC-1，會發生甚麼問題呢？

由於 PVC-1 綁定的 PV-1 在 node01 上，而 nginx-pod 又被指定到 node02，因此會造成衝突而無法執行。

因此，我們使用 storageClass 的 waitForFirstConsumer 來延遲 PV 與 PVC 的綁定時間。當 nginx-pod 使用了 PVC-1，schduler 會在同一個 storageClass 中尋找其他合適的 PV，例如 PVC-2、PVC-3，找到後才會綁定，這樣 Pod 就能跑起來。

總之，waitForFirstConsumer 的作用就是讓 PV 與 PVC 的綁定時間延遲，讓 Pod 與 PVC 綁定後，再去尋找合適的 PV。

---

* 建立 local PV：

```yaml
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /test
  nodeAffinity: # 指定 PV 要建立在哪個 Node 上
    required:
      nodeSelectorTerms:
      - matchExpressions: # 使用 Node 的 Label 篩選
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01
```
```bash
kubectl apply -f local-pv.yaml
```

* 檢查 PV 的狀態：

```bash
kubectl get pv local-pv
```
```text
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv   10Gi       RWO            Delete           Available           local-storage   <unset>                          3s
```

* 建立 PVC：

```yaml
# local-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
```
```bash
kubectl apply -f local-pvc.yaml
```
* 檢查 PVC 的狀態：

```bash
kubectl get pvc local-pvc
```
```text
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
local-pvc   Pending                                      local-storage   <unset>                 2s

```
> 因為還沒有任何 Pod 使用 PVC，所以 PVC 的狀態是 Pending 

* 建立 Pod：

```yaml
# nginx-local.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-local
spec:
  volumes:
  - name: local-vol
    persistentVolumeClaim:
      claimName: local-pvc
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: local-vol
```
```bash
kubectl apply -f nginx-local.yaml
```

> 等 Pod 跑起來後再次檢查 PVC 的狀態，就會發現已經被綁定了。

* 檢查 Pod 是否掛載成功：

```bash
kubectl exec -it nginx-local -- cat /usr/share/nginx/html/test.txt
```
```text
testing local pv
```

以上為 local PV 的使用方式，能夠確保需要特定資料的 Pod 一定會被分配到有 local PV 的 Node 上，解決了 hostPath 的問題。

### NFS StorageClass (Optional)

> 以下實作如果在 killercoda 上執行，可能會遇到一些權限問題，建議在自己的虛擬機上操錯。

前面雖然建立了一個 local-storage 的 storageClass，但由於使用「kubernetes.io/no-provisioner」作為 provisioner，因此只能達到「分類」、延遲綁定的效果，它只是一個「邏輯上」的 storageClass，並不能像前面介紹的那樣自動建立 PV。

因此這裡我們來實際建立一個功能完整的 storageClass：使用 nfs 作為 storageClass 的 provisioner。

這裡選用 [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) 作為 storageClass 的 provisioner，所以我們得先建置一個 nfs server：

> 如果不清楚 nfs 的概念，可以參考[這裡](https://linux.vbird.org/linux_server/centos6/0330nfs.php)

* 在 Master Node 安裝 nfs server 與 nfs client：
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

* 在**所有**的 Node 安裝 nfs client：
```bash
sudo apt install -y nfs-common
```

---
>**注意**

如果你檢查剛剛安裝的 nfs-common 是 inactive (dead)，如下：
  
```bash
systemctl status nfs-common
```
```text
● nfs-common.service
     Loaded: masked (Reason: Unit nfs-common.service is masked.)
     Active: inactive (dead)
```

先 unmask nfs-common：
```bash
sudo systemctl unmask nfs-common
```

如果 nfs-common 還是顯示 masked， 這有可能因為 nfs-common 的服務檔被連結到 /dev/null 了，如下：
```bash
file /lib/systemd/system/nfs-common.service
```
輸出：
```text
/lib/systemd/system/nfs-common.service: symbolic link to /dev/null
```

所以刪除連結檔，重啟服務即可：
```bash
sudo rm /lib/systemd/system/nfs-common.service
sudo systemctl daemon-reload
sudo systemctl restart nfs-common
```
最終要確定 nfs-common 是 active (running) 狀態：
```bash
systemctl status nfs-common
```
```text
● nfs-common.service - LSB: NFS support files common to client and server
     Loaded: loaded (/etc/init.d/nfs-common; generated)
     Active: active (running) since Fri 2024-05-24 15:18:15 UTC; 12s ago
```
***


* 回到 Master Node 上，建置一個要分享的目錄：
```bash
sudo mkdir -p /data/k8s
sudo chown nobody:nogroup /data/k8s
sudo chmod 777 /data/k8s
```

* 查看主機所在網域：
```bash
hostname -I
```
輸出：
```text
192.168.132.1
```

* 將要分享的目錄加入到「/etc/exports」：
```bash
echo -e "/data/k8s\t192.168.132.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
```

* 分享目錄：
```bash
sudo exportfs -av
```
輸出：
```text
exporting 192.168.132.0/24:/data/k8s
```

* 重啟 nfs server：
```bash
sudo systemctl restart nfs-kernel-server
```

* 查看是否有成功分享目錄：
```bash
showmount -e localhost
```
```text
Export list for localhost:
/data/k8s 192.168.132.0/24
```

測試一下分享目錄的效果：

* 建立分享目錄的掛載點：
```bash
sudo mkdir -p /mnt/nfs
```

* 掛載分享目錄：
```bash
sudo mount 192.168.132.1:/data/k8s /mnt/nfs
```

* 建立一個檔案到分享目錄：
```bash
touch /data/k8s/test.txt
```

* 可以在掛載點上看到 test.txt:
```bash
ls /mnt/nfs
```
```text
test.txt
```

完成 nfs server 的建置後，接下來透過 helm 安裝 「nfs-subdir-external-provisioner」:

> 什麼是「helm」？可以參考 [Day 11](https://ithelp.ithome.com.tw/articles/10346850)

* 安裝 helm：
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
``` 

* 加入 nfs-subdir-external-provisioner 的 repo：
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```

* 安裝 nfs-subdir-external-provisioner：
```bash
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.132.1 \
    --set nfs.path=/data/k8s \
    --set storageClass.name=nfs-storage
```
```text
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Fri May 24 13:13:22 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

* 查看 storageClass：
```bash
kubectl get sc nfs-storage
```
```text
NAME                	PROVISIONER                                 	RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete      	Immediate       	true               	   17m                 
```

* 查看 provisioner 是否有跑起來：
```bash
kubectl get po | grep nfs
```
```text
NAME                                           	   READY   STATUS 	 RESTARTS   AGE
nfs-subdir-external-provisioner-6444d75b85-56sts   1/1 	   Running   0      	17m
```
> 如果一直卡在 ContainerCreating，但沒有 Error 的 events，可以嘗試重新安裝 nfs-subdir-external-provisioner。如果還是不行，卡在了掛載相關的錯誤，請確認每台 Node 上的 nfs-common 為 active 的狀態。

* 建立一個 PVC：
```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-sc
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
```bash
kubectl apply -f pvc.yaml
```

* 可以看到建立 PVC 後，狀態馬上變成 Bound：
```bash
NAME   	   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound	test-pv                                	   500Mi  	  RWO        	 just-test  	<unset>             	14m
test-sc	   Bound	pvc-b5e66f93-7c2e-4d8b-84af-c507bf2f0829   100Mi  	  RWO        	 nfs-storage	<unset>             	2s
```

* storageClass 建立的 PV 在這裡：
```bash
kubectl get pv
```
```text
NAME   	   STATUS   VOLUME                                 	   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound	test-pv                                	   500Mi  	  RWO        	 just-test  	<unset>             	14m
test-sc	   Bound	pvc-b5e66f93-7c2e-4d8b-84af-c507bf2f0829   100Mi  	  RWO        	 nfs-storage	<unset>             	2s
```

* 刪除 PVC：
```bash
kubectl delete pvc test-sc
```

* 因為 storageClass 的 ReclaimPolicy 是 Delete，所以刪除 PVC 後，PV 也會被刪除：
```bash
kubectl get pv
```
```text
NAME  	  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          	STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
test-pv   500Mi  	 RWO        	Retain       	 Bound	  default/test-pvc  just-test  	   <unset>                      	20m
```

這就是 storageClass 的完整效果：使用者只需要建立 PVC，storageClass 會負責管理 PV 的建立與刪除。

### 今日小結

今天我們介紹了 PV、PVC 和 storageClass 的使用，PV 與 PVC 能把儲存空間從 Pod 獨立出來，在資料可以持續保存的情況下，讓 Pod 能夠更加靈活的使用儲存空間。而 storageClass 則能會自動管理 PV 的建立與刪除，讓我們只需建立 PVC，就能拿到儲存空間與達到「分類」管理的效果。

今天就是「Storage」章節的最後一篇，明天我們將進入「Workloads & Scheduling」章節，來談談該如何調度 Pod 以及資源管理。

-----
**參考資料**

[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

[Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)

[Volumes---local](https://kubernetes.io/docs/concepts/storage/volumes/#local)

[How To Set Up an NFS Mount on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04)

[Kubernetes NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

[How to Setup Dynamic NFS Provisioning in a Kubernetes Cluster](https://hbayraktar.medium.com/how-to-setup-dynamic-nfs-provisioning-in-a-kubernetes-cluster-cbf433b7de29)

[透過 NFS Server 在 K3s cluster 新增 Storage Class](https://zhengwei-liu.medium.com/%E9%80%8F%E9%81%8E-nfs-server-%E5%9C%A8-k3s-cluster-%E6%96%B0%E5%A2%9E-storage-class-eeae09141005)

[解决nfs-common.service is masked 问题](https://zhuanlan.zhihu.com/p/469398833)

