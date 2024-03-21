# storage -- PV、PVC and StorageClass

上個章節介紹了`volume`，但如果我們只是單純的使用`volume`會有以下幾個缺點:

  * `volume`的生命週期和`pod`是一樣的，當`pod`被刪除時，`volume`也會被刪除
  
  * 假如定義了hostPath，當pod轉移到另一個node上時，我們無法保證新的node上也有同樣的「hostPath」

  * 每個pod都要定義自己的volume，相當麻煩。


如果能先建立一個獨立於pod之外能持續存活的「volume」，等pod有需要時再依需求分配，這樣就方便許多了。而這樣的「volume」就是`Persistent Volume`(PV)，而pod對`PV`的需求就透過`Persistent Volume Claim`(PVC)來提出。

而`PV`的配置方式可以分為以下兩種模式:

 * Static: 由管理者自行定義並建置

 * Dynamic: 由`StorageClass`自動建置

而`storageClass`也需要管理者來配置，那既然都要手動配置了，為什麼不直接配置`PV`呢？ 原因是，`storageClass`可以讓`PV`的配置更加彈性，例如: 當PV被用完時，`storageClass`收到新的PVC後能自動建立新的PV  

## Persistent Volume (PV) and Persistent Volume Claim (PVC)

我們先來看看`PV`的yaml格式:

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

 * `capacity`: 定義`PV`的大小，例如: `1Gi`、`500Mi`等

 * `accessModes`: 定義`PV`的存取模式，有以下三種:
 
   * **ReadWriteOnce**(RWO): 只能被**單一**node掛載為讀寫模式

   * **ReadOnlyMany**(ROX): 可以被**多個**node掛載為唯讀模式

   * **ReadWriteMany**(RWX): 可以被**多個**node掛載為讀寫模式


 * `persistentVolumeReclaimPolicy`: 定義當`PVC`被刪除時的行為，有以下兩種:
 
   * Retain: 保留`PV`裡的資料，除非被管理員手動刪除，不然不能再被PVC使用

   * Delete: 直接刪除`PV`

   * > 本來應該還有一個`Recycle`，但已經被棄用了

 * `pv-type`: 定義`PV`的類型，例如: `nfs`、`hostPath`等。其他的`pv-type`可參考[官網](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)

 * `pv-type-configurations`: 根據不同的`pv-type`而有不同的配置，例如: `nfs`需要定義`server`、`path`等

> 以上只是最基本的配置，其他的可以參考[官網](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)

 我們再來看看`PVC`的yaml格式:

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

* `accessModes`: 同`PV`的`accessModes`

* `resources`: 定義所需的大小，例如: `1Gi`、`500Mi`等

另外，也可以使用`selector`來指定`PV`:

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

當建立`PVC`後，K8s會自動尋找符合條件的PV，如果符合條件的PV不存在，則`PVC`會一直處於`Pending`狀態，直到符合條件的PV被建立。

會拿來比較的條件例如:
  
  * `accessModes`是否符合

  * PV的`capacity`是否大於或等於`PVC`的`request`

  * `selector`是否符合

  * `storageClass`是否符合

**注意**

`PV`與`PVC`是一對一關係，`PV`一旦和某個`PVC`綁定後，就不能再被其他`PVC`使用。

## StorageClass

`storage class`的yaml格式如下:

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

* `provisioner`: `storageClass`會建立`PV`，就需要一個`PV`的「提供者」。提供者的清單可參考[官網](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)

* `reclaimPolicy`: 同`PV`的`ReclaimPolicy`，預設為`Delete`

* `allowVolumeExpansion`: 是否允許`PVC`要求更多的size，`true`為允許。

* `volumeBindingMode`: 定義`PV`的綁定模式，有以下兩種:

  * **Immediate**: 當`PVC`被建立時，`storageClass`會立即建立`PV`

  * **WaitForFirstConsumer**: 當`PVC`被建立時，`storageClass`不會立即建立`PV`，而是等到有`pod`使用`PVC`時才建立`PV`

> 以上只是最基本的配置，其他的可以參考[官網](hhttps://kubernetes.io/docs/concepts/storage/storage-classes/)

## PV與PVC的使用

> 先準備一個檔案，建立pod後看看是否有成功掛載

* 建立測試用的檔案:
    
```bash
echo "testing pv and pvc" > /tmp/test.txt
```

* 接著建立一個`PV`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
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

**Tips**

這個storageClassName是我們隨便取的，不用**真的**建立一個storage class。

因為如果不指定`storageClassName`，則會使用系統預設的storage class，這樣後面PVC與Pod建立後，用的就不會是我們自訂的PV，而是預設storage class給的PV

所以使用「編造storageClassName」的技巧，就能繞開預設的storage class，讓我們自己定義的PV被使用。


* 建立後檢查`PV`的狀態:
```bash
$ kubectl get pv test-pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
test-pv   500Mi      RWO            Retain           Available           just-test      <unset>                          3s

```
> STATUS為`Available`，表示`PV`還沒被使用，可以被`PVC`請求

* 再建立一個`PVC`:

```yaml
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


* 檢查`PVC`的狀態:
```bash
$ kubectl get pvc test-pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound    test-pv   500Mi      RWO            just-test      <unset>                 3s
```

* 建立一個Pod，掛載`PVC`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
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

建立後查看一下是否掛載成功:
```bash
kubectl exec -it nginx -- cat /usr/share/nginx/html/test.txt
```
如果掛載有成功，輸出如下:
```txt
testing pv and pvc
```


