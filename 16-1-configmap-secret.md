# ALM: configMap & secret

除了前面章節介紹的例子，「環境變數(environment variables)」也是前置作業中的常見設定，而環境變數由一對`key-value`組成，例如`USER=root`。在`yaml`中可以這樣定義環境變數:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - command:
    - sleep
    - "300"
    env: # 環境變數
    - name: USER
      value: michael
    image: busybox
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

如果要定義多個環境變數，就在`spec.containers.env`中加入多個`key-value`即可，例如:
```yaml
...
...(省略)
containers:
- command:
  - sleep
  - "300"
  env: # 環境變數
  - name: USER
    value: michael
  - name: PASSWORD
    value: 123456
...
...(省略)
```

但是一旦需要的環境變數開始變多，這樣設定非常沒有效率。且這樣的設定方式也不易閱讀。以上面的例子為例，如果能夠把`USER`和`PASSWORD`一起放在`user-data-config`中，再直接引入`pod`中，是不是會比較方便呢? 這時候就可以使用`configMap`和`secret`來達成這個目的。

除了環境變數外，`configMap`和`secret`也可以用來設定「command-line 參數」、存放「設定檔」等等。

> configMap與secret通常會與volumn搭配使用，相關操作會在後續章節介紹

## ConfigMap

> 提醒: configMap不是用來存放大量資料的，所以configMap的資料輛不能超過1MB。

建立`configMap`的方式如下:

* 存放key-value類型的data:
```bash
kubectl create configmap <configmap-name> --from-literal=<key>=<value>
```

* 存放檔案類型的data:
```bash
kubectl create configmap <configmap-name> --from-file=<file-path>
```

舉例而言，如果要建立一個名為`user-data-config`的`configMap`，裡面有`USER`和`PASSWORD`兩個環境變數，可以這樣建立:
```bash
kubectl create configmap user-data-config-michael --from-literal=USER=michael --from-literal=PASSWORD=123456
```

建立configMap後，可以使用`describe`指令查看:
```bash
kubectl describe configmap user-data-config-michael
```
```yaml
Name:         user-data-config-michael
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
PASSWORD:
----
123456
USER:
----
michael

BinaryData
====

Events:  <none>
```

或者我們先準備好一個檔案，例如`user-data-config.env`:
```bash
echo -e "USER: Bob\nPASSWORD: 123456" > user-data-config.env
```

然後使用`--from-file`的方式建立`configMap`:
```bash
kubectl create configmap user-data-config-bob --from-file=user-data-config.env
```
> 如此一來，`configMap`存放的就是`user-data-config.env`這個檔案與其內容。

建立`configMap`後，可以使用`describe`指令查看:
```bash
kubectl describe configmap user-data-config-bob
```
```yaml
Name:         user-data-config-bob
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
user-data-config.env: # 注意到這裡多了一個檔名
----
USER: Bob
PASSWORD: 123456


BinaryData
====

Events:  <none>
```

> 當然，你也可以使用`yaml`檔案來建立`configMap`，例如:


* 存放「key-value」類型的data:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-data-config-alice
data:
    USER: "Alice" # 記得加上引號
    PASSWORD: "123456"
```

* 存放「檔案」類型的data:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-data-config-alice
data:
  alice-data: |
    USER=Alice 
    PASSWORD=123456
```
> alice-data是檔案的名稱，USER=Alice和PASSWORD=123456是檔案的內容


## 使用 ConfigMap

建立好`configMap`後，就可以在`pod`的中引入，而引入的方式分兩種:
  
#### 1. **引入全部**: 將整份`configMap`引入`pod`中

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: all-config
spec:
  containers:
  - command:
    - sleep
    - "300"
    envFrom: # 引入configMap
    - configMapRef:
        name: user-data-config-michael
    image: busybox
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

**測試**

* 建立pod:
```bash
kubectl apply -f all-config.yaml
```

* 等待pod建立完成後，透過`kubectl exec`列出pod的環境變數:
```bash
kubectl exec -it all-config -- env | grep 'USER\|PASSWORD'
```
輸出:
```text
PASSWORD=123456
USER=michael
```

#### 2. **引入部分**: 只引入`configMap`某些`key`的`value`

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: part-config
spec:
  containers:
  - command:
    - sleep
    - "300"
    env: # 引入configMap的部分key
    - name: USER
      valueFrom:
        configMapKeyRef:
          name: user-data-config-michael
          key: USER # 只要USER的value
    image: busybox
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

**測試**

* 建立pod:
```bash
kubectl apply -f part-config.yaml
```

* 等待pod建立完成後，透過`kubectl exec`列出pod的環境變數:
```bash
kubectl exec -it part-config -- env | grep 'USER\|PASSWORD'
```
輸出:
```text
USER=michael
```
> 可以看到，只有`USER`的環境變數被引入

#### 在指令中引入`ConfigMap`作為參數

這算是環境變數的延伸應用，把環境變數作為指令的參數，例如:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: arg-config
spec:
  containers:
  - command: ["echo", "$(USER)"]
    env:
    - name: USER
      valueFrom:
        configMapKeyRef:
          name: user-data-config-michael
          key: USER
    image: busybox
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

**測試**

* 建立pod:
```bash
kubectl apply -f arg-config.yaml 
```
> 這裡pod執行完echo後就會結束，所以如果你看到非running的狀態，不用擔心

* pod已經完成了它的任務，我們能透過`kubectl logs`指令查看結果:
```bash
kubectl logs arg-config
```
輸出:
```text
michael
```
## Secret

`Secret`和`ConfigMap`很像，都可以用來存放「key-value」或「檔案內容」，不過`Secret`是用來存放較為機密資訊，例如密碼、金鑰等等。

> 比如上面的例子中，直接將`PASSWORD`寫在`configMap`中是很不安全的。

建立`Secret`的方式和`configMap`很像，可以使用`--from-literal`或`--from-file`的方式建立:

```bash
kubectl create secret generic user-data-secret-1 --from-literal=PASSWORD=123456
```

或者先準備好一個檔案。例如`user-data-secret.env`，不過將`value`轉換成`base64`格式:

* 例如，將`123456`轉換成`base64`:
```bash
echo -n "123456" | base64
```
轉成base64的輸出:
```text
MTIzNDU2
```

* 然後將`base64`的結果寫入`user-data-secret.env`:
```bash
echo "password: MTIzNDU2" > passwd.conf 
```

* 最後同樣用`--from-file`建立:
```bash
kubectl create secret generic user-data-secret-2 --from-file=passwd.conf
```

使用`describe`指令查看`secret`，會發現`value`的訊息已經被隱藏了:
```bash
kubectl describe secret user-data-secret-1 user-data-secret-2
```
```yaml
Name:         user-data-secret-1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
PASSWORD:  6 bytes


Name:         user-data-secret-2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
passwd.conf:  19 bytes
```
> 不過，如果使用`kubectl get secret <secret> -o yaml`指令，還是可以看到「data」，所以並不絕對安全。

同樣的，你也可以用`yaml`建立`secret`:

* 存放「key-value」類型的data:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-data-secret-3
data:
  PASSWORD: MTIzNDU2 # 使用base64編碼的"123456"
```

* 存放「檔案」類型的data:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-data-secret-4
data:
  passwd.conf: |
    MTIzNDU2
```

## 使用 Secret

在`pod`中引入`secret`的方式和`configMap`很像，也分兩種:

[參考](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#define-container-environment-variables-using-secret-data)

#### 1. **全部引入**: 將整份`secret`引入`pod`中

```yaml
# 同樣於spec.containers.envFrom中引入
envFrom: 
- secretRef:
    name: user-data-secret -1
```
#### 2. **部分引入**: 只引入`secret`某些`key`的`value`

```yaml
# 同樣於spec.containers.env中引入
env: 
- name: PASSWORD
  valueFrom:
    secretKeyRef:
      name: user-data-secret-1
      key: PASSWORD
```

#### Secret的安全性

其實不難發現，僅用`base64`來編碼敏感資料是不夠的，可以輕鬆地被反推解碼:
```bash
$ echo "MTIzNDU2" | base64 -d
123456
```

所以單單使用`secret`並不能有效的保護敏感資料，必須搭配其安全機制，例如後續會介紹的`RBAC`、`靜態加密`等等，這裡提供相關的參考資料:
 * [Kubernetes官方文件](https://kubernetes.io/docs/concepts/configuration/secret/)
 * [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
 