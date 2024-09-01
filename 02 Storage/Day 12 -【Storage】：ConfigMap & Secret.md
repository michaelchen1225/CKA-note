# Day 12 -【Storage】：ConfigMap & Secret

### 今日目標

* 了解 configMap 與 Secret 的用途
* 建立 configMap、Secret
  * 存放 key-value 或檔案
  * 用 Secret 存放 private image registry 的登入資訊
* 在 Pod 中引入 configMap、secret 做為環境變數或指令參數
* 在 Pod 中使用 Secret 來拉取 private registry 的 image

我們在 [Day 5](https://ithelp.ithome.com.tw/articles/10345967) 曾經介紹過 Pod 中的環境變數，如果要定義多個環境變數，就在 `spec.containers.env[]` 中加入多個 key-value 即可，例如：
```yaml
......
containers:
- command:
  - sleep
  - "300"
  env: # 環境變數
  - name: USER
    value: michael
  - name: PASSWORD
    value: 123456
......
```

但是一旦需要的環境變數開始變多，這樣設定非常沒有效率且不易閱讀。

以上面的例子為例，如果能夠把 USER 和 PASSWORD 事先儲存起來，再引入 Pod 裡，是不是會比較方便呢？ 

這時候就可以使用 configMap 和 secret 來達成這個目的。除了環境變數外， configMap 和 secret 也可以用來設定「command-line 參數」、存放「檔案」等等。

(configMap 與 secret 可以和 volumn 搭配使用，不過這部分會等到明天介紹 Volume 時再提到)

### ConfigMap

ConfigMap 是一種 K8s 物件，用來存放「key-value」或「檔案」，然後可以在 Pod 中引入這些資料。這樣設計的好處是，能將較為彈性的資料與設定檔從 Pod 中獨立出來，當 Pod 要部署到不同環境時就不用修改程式碼，只需替換 ConfigMap 即可。

> 提醒：configMap 不是用來存放大量資料的，所以 configMap 的資料輛不能超過 1 MB。

建立 configMap 的方式如下：

1. 存放 key-value 類型的 data：
```bash
kubectl create configmap <configmap-name> --from-literal=<key>=<value>
```

2. 存放檔案類型的 data：
```bash
kubectl create configmap <configmap-name> --from-file=<file-path>
```

舉例而言，如果要建立一個名為 user-data-config 的 configMap，裡面存放「使用者的相關變數」，可以這樣建立:

1. **用「 key-value 」的方式存放於 configMap**：

* 建立一個名為「user-data-config-michael」的 configMap，裡面存放兩組 key-value 資料：USER=michael 和 PASSWORD=123456：
```bash
kubectl create configmap user-data-config-michael --from-literal=USER=michael --from-literal=PASSWORD=123456
```

* 建立 configMap 後，可以使用 describe 指令查看：
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

2. **用「檔案」的方式存放於 configMap**：

* 我們先準備好一個檔案，例如 user-data-bob.conf :

```bash
echo -e "# this is Bob's data\nUSER=Bob\nPASSWORD=123456" > user-data-bob.conf
```

* 然後使用 --from-file 的方式建立 configMap，用來儲存 user-data-bob.conf 這個檔案：
```bash
kubectl create configmap user-data-config-bob --from-file=user-data-bob.conf
```

* 建立 configMap 後，可以使用 describe 指令查看：
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
user-data-config-bob.env:
----
# this is Bob's data
USER=Bob
PASSWORD=123456


BinaryData
====

Events:  <none>
```

> 當然，你也可以使用 yaml 檔案來建立 configMap，例如：


* 存放「key-value」類型的 data：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-data-config-alice
data:
    USER: "Alice" # value 記得加上引號
    PASSWORD: "123456"
```

* 存放「檔案」類型的data：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-data-config-jack
data:
  Jack-data.env: | 
    # this is Jack's data
    USER=Jack
    PASSWORD=123456
```
> Jack-data 是檔案的名稱，「|」後就是檔案內容。


### 在 Pod 中 使用 ConfigMap 的 key-value 作為環境變數

建立好帶有 key-value 資料的 configMap 後，就可以在 Pod 中引入作為環境變數，而引入的方式分兩種：
  
1. **configMapRef**：將整份 configMap 的 Key value 引入 Pod 中

```yaml
# all-config.yaml
apiVersion: v1
kind: Pod
metadata:
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
```

**測試**

* 建立 Pod:
```bash
kubectl apply -f all-config.yaml
```

* 等待 Pod 建立完成後，透過 kubectl exec 列出 Pod 的環境變數：
```bash
kubectl exec -it all-config -- env | grep 'USER\|PASSWORD'
```
```text
PASSWORD=123456
USER=michael
```

2. **configMapKeyRef**：只引入 configMap 部分的 key-value

```yaml
# part-config.yaml
apiVersion: v1
kind: Pod
metadata:
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
          key: USER # 只要USER這組 key-value
    image: busybox
    name: busybox
```

**測試**

* 建立 Pod :
```bash
kubectl apply -f part-config.yaml
```

* 等待 Pod 建立完成後，透過 kubectl exec 列出 Pod 的環境變數:
```bash
kubectl exec -it part-config -- env | grep 'USER\|PASSWORD'
```
輸出:
```text
USER=michael
```
> 只有 USER 的環境變數被引入

### 在指令中引入 ConfigMap 作為參數

這算是環境變數的延伸應用，把環境變數作為指令的參數，例如：

```yaml
# arg-config.yaml
apiVersion: v1
kind: Pod
metadata:
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
```

**測試**

* 建立 Pod :
```bash
kubectl apply -f arg-config.yaml 
```
> 這裡 Pod 執行完 echo 後就會結束，結束後就不會是 running 的狀態

* Pod 已經完成了它的任務，我們能透過 kubectl logs 查看輸出結果：
```bash
kubectl logs arg-config
```
輸出:
```text
michael
```

> 還有一種用法是將 configMap 中的「檔案」引入，這個部分等到明天介紹 volume 時會再提到。

### Secret

Secret 和 ConfigMap 很像，都可以用來存放「key-value」或「檔案內容」，不過 Secret 是用來存放較為機密資訊，常見用途有：

* 設定 Pod 的環境變數或引入檔案
* 存放機密資訊，例如 SSH keys、密碼等等
* 存放 private image registry 的登入資訊 

根據用途的不同，secret 也分為不同的類型，可以在 secret yaml 的「type」欄位指定。以下介紹兩種常見的類型：

* **Opaque**：預設的 secret 類型，用來存放任意資料，不過建議把敏感資料轉換成 base64 格式存放。

* **Docker config Secrets**：如果某個 Pod 需要從 private registry 下載 image，就需要用到這個 secret 類型來提供登入資訊。

> 關於其他的 secret 類型，可以參考 [官網](https://kubernetes.io/docs/concepts/configuration/secret/)。

### 建立 Opaque Secret

建立 Opaque Secret 的方式和 configMap 也很像，可以使用「--from-literal」或「--from-file」的方式建立：

```bash
kubectl create secret generic user-data-secret-1 --from-literal=PASSWORD=123456
```

或者先準備好一個檔案。例如 user-data-secret.env，這裡我們將 value 先轉換成 base64 格式：

* 例如，將 123456 轉換成 base64 :
```bash
echo -n "123456" | base64
```
```text
MTIzNDU2
```

* 然後將 base64 的結果寫入 user-data-secret.env :
```bash
echo "password: MTIzNDU2" > passwd.conf 
```

* 最後用「--from-file」建立；
```bash
kubectl create secret generic user-data-secret-2 --from-file=passwd.conf
```

使用 describe 指令查看 secret，會發現 value 的訊息已經被隱藏了：
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
> 不過，如果使用 kubectl get secret \<secret-name> -o yaml 指令，還是可以看到「data」，所以並不絕對安全，這在今天的最後會補充說明。

你也可以用 yaml 建立 Opaque Secret：

* 存放「key-value」類型的 data：

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
> passwd.conf 是檔案的名稱，「|」後面就是檔案的內容

### 在 Pod 中 使用 Opaque Secret 的 key-value 作為環境變數

在 Pod 中引入 secret 的方式和 configMap 很像，也分兩種：

1. **secretRef**：將整份 secret 的 key-value引入 Pod 中

```yaml
# 同樣於 spec.containers.envFrom 中引入
envFrom: 
- secretRef:
    name: user-data-secret -1
```

2. **secretKeyRef**：只引入 secret某些 key 的 value

```yaml
# 同樣於 spec.containers.env 中引入
env: 
- name: PASSWORD
  valueFrom:
    secretKeyRef:
      name: user-data-secret-1
      key: PASSWORD
```

### 建立 Docker config Secrets

Docker config Secrets 是用來存放 private registry 的登入資訊，可以用指令直接建立：

```bash
kubectl create secret docker-registry michael-docker \
  --docker-email=michael@mail.example \
  --docker-username=michael \
  --docker-password=123456 \
  --docker-server=my-registry.example:5000
```

建立後，我們就能在 Pod 中引入這個 secret 作為登入資訊，來 pull private registry 的 image：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-test
spec:
  containers:
  - name: private-reg-test
    image: <private-image>
  imagePullSecrets:
  - name: michael-docker # secret 的名稱
```

### Secret 的安全性

其實不難發現，僅用 base64 來編碼敏感資料是不夠的，可以輕鬆地被反推解碼：
```bash
echo "MTIzNDU2" | base64 -d
123456
```

所以單單使用 secret 並不能有效的保護敏感資料，必須搭配其安全機制，例如 RBAC、靜態加密等等，這裡提供相關的參考資料：

 * [Secret](https://kubernetes.io/docs/concepts/configuration/secret/)

 * [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)


### 今日小結

今天介紹了 configMap 和 secret，兩者皆可用存放環境變數或檔案，不過 secret 用來存放機密資訊，例如密碼、SSH keys 等等。透過 configMap 和 secret，我們可以將 Pod 的設定與資料獨立出來，讓 Pod 更具彈性。

另外，configMap 和 secret 中儲存的檔案可以透過 volume 的方式掛載到 Pod 中，這部分會在明天介紹。

-----
**參考資料**

* [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

* [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-a-container-environment-variable-with-data-from-a-single-configmap)

* [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)

* [Secret](https://kubernetes.io/docs/concepts/configuration/secret/)

* [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

* [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)