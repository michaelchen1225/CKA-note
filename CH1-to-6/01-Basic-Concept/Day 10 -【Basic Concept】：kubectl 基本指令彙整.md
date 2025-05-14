## 【Basic Concept】：kubectl 基本操作彙整

## 目錄

* [物件類別縮寫](#物件類別縮寫)

* [常用指令整理 --- get & describe](#常用指令整理-----get--describe)

* [常用指令整理：Pod](#常用指令整理pod)

* [常用指令整理：Deployment](#常用指令整理deployment)

* [常用指令整理：Rollout history、Rollback](#常用指令整理rollout-historyrollback)

* [常用指令整理：Service](#常用指令整理service)

* [kubectl 的小技巧](#kubectl-的小技巧)

* [修改現有物件(一)：kubectl edit](#修改現有的物件一kubectl-edit)

* [修改現有物件(二)：kubectl patch](#修改現有的物件二kubectl-patch)



今天是「Basic Concept」章節的倒數第二篇，我們在前面幾天介紹了 Kubernetes 中的基本物件，其中大量的操作都是透過 kubectl 這個指令來完成的，所以管理者最好能快速且熟練的使用 kubectl。

因此，今天將針對前面幾天介紹的範圍，也就是 Pod 、Deployment、Service、Namespace，整理出一些常用的指令與技巧。

**提醒**

> 要在 kubectl 中指定 Namespace，加入「-n」參數即可，例如：「kubectl -n kube-system get po」，底下就不特別整理了。

### 物件類別縮寫

在 kubectl 指令中，我們常常需要指定資源類別，而縮寫可以大大加快下指令的速度，以下是常見的縮寫：

物件類別|縮寫
---|---
pod|po
deployment|deploy
service|svc
namespace|ns

* 例如要列出所有的 Service，以前我們會下達「kubectl get service」，而搭配縮寫後，只要敲：
```bash
kubectl get deploy
```

你也可以用以下指令，查看其他資源的縮寫：
```bash
kubectl api-resources
```

### 常用指令整理 --- get & describe


* 列出**所有**資源的清單，不論是 Pod 、deployment、service 等等：
```bash
kubectl get all
```

* 列出某種資源的清單：
```bash
kubectl get <object-type>
```

* 列出特定資源：
```bash
kubectl get <object-type> <object-name>
```

* 列出**所有** Namespace 中，某種資源的清單：
```bash
kubectl get <object-type> -A
```

* 列出整個 cluster 中**所有**的資源：
```bash
kubectl get all -A
```

* 計算某種資源的個數：
```bash
kubectl get <object-type> | wc -l
# 結果要記得減1，因為第一行是 header
```
或是:
```bash
kubectl get <object-type> --no-headers | wc -l
```

* 查看特定資源的 yaml 格式：
```bash
kubectl get <object-type> <object-name> -o yaml
```
> "-o" 可以指定輸出的格式，例如：json、yaml、wide 等等


* 查看某種資源的 label：
```bash
kubectl get <object-type> --show-labels
```

* 使用 selector 篩選特定資源：
```bash
kubectl get <object-type> -l <key=value>
# 多個label用逗號隔開，例如: -l <key1=value1,key2=value2>
```

* 搭配 jsonpath 查看特定欄位：
```bash
kubectl get <object-type> -o jsonpath=<jsonpath-expression>
```

> 關於 jsonpath 的介紹與使用，可以參考[附錄](https://github.com/michaelchen1225/CKA-note/blob/main/%E9%99%84%E9%8C%84/%E9%99%84%E9%8C%841-jsonpath.md)

* 查看特定資源的詳細資訊：
```bash
kubectl describe <object-type> <object-name>
```

### 常用指令整理：Pod 

* 建立一個 Pod：
```bash
kubectl run <pod-name> --image <image-name>
```

* 建立一個 Pod 並指定容器開放的 port：
```bash
kubectl run <pod-name> --image <image-name> --port <port-number>
```

* 建立一個 Pod ，並順便幫它建立一個 ClusterIP service：
```bash
kubectl run <pod-name> --image <image-name> --port <port-number> --expose
# service的名字會與pod相同，port會與pod的port相同
```

* 建立一個 Pod ，並指定 label：
```bash
kubectl run <pod-name> --image <image-name> -l <key=value>
# 如果有多個label，可以用逗號隔開，例如: -l <key1=value1,key2=value2>
```

* 更新 Pod 的 image：
```bash
kubectl set image pod <pod-name> <container-name>=<new-image-name>
# 若 Pod 是用 kubeclt run 建立，則 container-name預設為 pod-name
```

* 查看 Pod 的 log：
```bash
kubectl logs <pod-name>
```

* 持續的輸出 Pod 的 log：
```bash
kubectl logs -f <pod-name>
```

* 查看特定 Pod 中 container 的 log：
```bash
kubectl logs <pod-name> -c <container-name>
```

* 查看 Pod 的 IP
```bash
kubectl get pod <pod-name> -o wide
```
或是：

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'
```

* 將本機的檔案複製到 Pod 中：
```bash
kubectl cp <local-file-path> <pod-name>:<pod-file-path>
```

* 將 Pod 中的檔案複製到本機：
```bash
kubectl cp <pod-name>:<pod-file-path> <local-file-path>
```

* 開啟一個 terminal 與 Pod 互動：
```bash
kubectl exec -it <pod-name> -- /bin/sh
```

* 在 Pod 中執行指令：
```bash
kubectl exec  <pod-name> -- <command>
```

* 查看 Pod 被放在哪個 node 上：
```bash
kubectl get pod <pod-name> -o wide
```

或是
```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'
```

### 常用指令整理：Deployment

* 建立一個 Deployment：
```bash
kubectl create deploy <deploy-name> --image <image-name> --replicas <replicas-number>
# 不指定replicas的話，預設為1
```

* 建立一個 Deployment 並指定容器開放的 port：
```bash
kubectl create deploy <deploy-name> --image <image-name> --replicas <replicas-number> --port <port-number>
```

* 改變 Deployment 的 replicas 數量：
```bash
kubectl scale deploy <deploy-name> --replicas <new-replicas-number>
```

* 更新 Deployment 的 image：
```bash
kubectl set image deploy <deploy-name> <container-name>=<new-image-name>
# 若 Deployment 是由 kubectl create 建立，則 container-name 預設為 deploy-name
```

### 常用指令整理：Rollout history、Rollback


* 查看 Rolling Update 的歷史：

```bash
kubectl rollout history deploy <deploy-name>
```

* 查看特定 Revision 的詳細資訊：
```bash
kubectl rollout history deploy <deploy-name> --revision=<revision-number>
```

* 回到上一個版本：
```bash
kubectl rollout undo deploy <deploy-name>
```

* 回到特定 Revision：
```bash
kubectl rollout undo deploy <deploy-name> --to-revision=<revision-number>
```

* 標記此次版本的 CHANGE-CAUSE：
```bash
kubectl annotate deploy <deploy-name> kubernetes.io/change-cause="<change-cause>"
```

* 重啟 Deployment：
```bash
kubectl rollout restart deploy <deploy-name>
```

* 查看 Deployment 的 rolling 狀態：
```bash
kubectl rollout status deploy <deploy-name>
```

### 常用指令整理：Service

* 為 pod/deploymnet 建立一個 service：
```bash
kubectl expose <object-type> <object-name> --port <port-number> --target-port <target-port-number> --name <service-name> --type <service-type>
# 不指定 --target-port，預設與 --port 相同
# 不指定 --type，預設為 ClusterIP
# 不指定 --name，預設為 object-name
```

* 建立一個 Service：
```bash
kubectl create service <service-type> <service-name> --tcp <port-number>:<target-port-number>
# 可以另外加上 --node-port <node-port-number> 來建立 NodePort Service
```

* 查看 service 的 log：
```bash
kubectl logs svc/<service-name>
```

---

### kubectl 的小技巧

1. 不要自己刻 yaml，善用「--dry-run=client -o yaml」來產生 yaml 樣本：

> 基本上只要用 kubectl 建立新的物件都可以用這個方法，例如用 kubectl expose --dry-run=client -o yaml 來產生 service 的 yaml 樣本。

2. 善用 -h 來查看指令的使用方式，例如：
```bash
kubectl -h
kubectl run -h
kubectl create -h
kubectl expose -h
...
```

3. 如果某個資源在刪除時一直卡在 Terminating 狀態，可以使用 --force --grace-period=0 來強制刪除：
```bash
kubectl delete <object-type> <object-name> --force --grace-period=0
```

4. 假如我們有一份新的 yaml 想取代某個舊有物件，可以使用 replace 指令：
```bash
kubectl replace -f <new-yaml-file> --force --grace-period=0
```
> 會先刪除舊物件，然後根據新的 yaml 建立一個新的物件。須注意的是，新 yaml 中的物件名稱與 Namespace 都要與舊的物件相同。

5. 將 kubectl 設定 alias 為「k」，並且善用 bash 的自動補全功能(Tab 鍵)，能大大提升下指令的速度。相關設定請參考 [Day 03](https://ithelp.ithome.com.tw/articles/10345660) 的「Tips 1：kubectl bash completion」。

### 修改現有的物件(一)：kubectl edit

如果某個資源已經被建立了，但是想要修改一些設定，可以使用 kubectl edit 來進行更新，其語法如下：

```bash
kubectl edit <resource_type>/<resource_name>
```

底下提供了兩個範例情境，去更新先有資源的 image 與 cpu request：

**範例情境1：更新 image**

* 部署一個 Deployment：
```bash
kubectl create deployment nginx --image nginx --replicas 3
```

* 使用「kubectl edit」來更新 image：

```bash
kubectl edit deploy nginx
```

* 這時會會進入文字編輯器，就和編輯 yaml 檔案一樣，我們將 image 改成 nginx:1.19.1：
```yaml
......
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx:1.19.1 # 改這裡
        resources: {}
......
```

* 儲存離開後，會看到系統提示：
```text
deployment.apps/nginx edited
```

* 查看更新後的 Deployment image：
```bash
kubectl describe deploy nginx | grep -i image
```

```text
 Image:         nginx:1.19.1
```

**範例情境2：更新 cpu request**

> CPU request 算是「Workloads & Scheduling」章節的內容，這裡先了解 kubectl edit 的用法，之後會再針對 Resource 的部分做進一步的介紹。

* 執行一個 Pod：

```bash
kubectl run nginx --image nginx
```

* 將 Pod 的 `cpu request` 設置為 700m：

```bash
kubectl edit po nginx
```
修改成以下:
```yaml
......
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    resources: 
      requests:
        cpu: "700m" # 改這裡
......
```

儲存離開後，會看到系統給出提示：
```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
# pods "nginx" was not valid:
# * metadata: Invalid value: "BestEffort": Pod QoS is immutable
# * spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
......
```

重點是這行：
```yaml
# * spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
```

也就是說，除了以下欄位外，其餘是不允許直接 edit 的：

  * spec.containers[*].image

  * spec.initContainers[*].image

  * spec.activeDeadlineSeconds

  * spec.tolerations (只允許新增)

  * spec.terminationGracePeriodSeconds (如果之前是負數，則允許設置為1)

> 上面出現的新名詞 tolerations 是 Sheduling 的重要概念，之後會在「Workloads & Scheduling」章節中介紹。 

不過，雖然直接 edit 行不通，不過我們離開文字編輯時，會發現系統提示：
```text
error: pods "nginx" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-3168862298.yaml"
error: Edit cancelled, no valid changes were saved.
```

也就是說，剛才 edit 的內容雖然不會生效，但副本被儲存到了「/tmp/kubectl-edit-3168862298.yaml」，我們可以使用以下方式將這個檔案重新套用：

```bash
kubectl replace -f /tmp/kubectl-edit-3168862298.yaml --force --grace-period=0
```

這會直接取代並刪除掉現有的 Pod，然後根據「/tmp/kubectl-edit-3168862298.yaml」重新建立一個新的Pod：

```bash
kubectl get po nginx
```
```text
NAME                     READY   STATUS    RESTARTS   AGE
nginx                    1/1     Running   0          10s
```

```bash
kubectl describe po nginx | grep -i cpu
```
```text
    cpu:     700m
```

從上面兩個範例來看，kubectl edit 在修改的欄位上會有些許限制，而修改不同的資源也會有不同的欄位限制，不過我們可以使用 kubectl replace 來取代舊有的物件，這樣就可以達到修改的目的。

當修改的經驗多了之後，你就會知道哪些欄位可直接用 edit 修改的，而哪些不行。如果你知道你要修改的欄位不能直接用 edit 更新，可以先將資源的 yaml 輸出，修改新 yaml 後再用 replace 來取代：


```bash
kubectl get <resource_type> <resource_name> -o yaml > /tmp/<resource_name>.yaml
```
(這樣可以節省掉因為 kubectl edit 失敗而要再次退出文字編輯器的時間，以及複製 /tmp/kubectl-edit-xxx 的步驟)

> 最後要提醒的是，以版本控制的角度來說，頻繁使用 kubectl edit 來修改資源可能會造成資源的版控不易追蹤，因為 edit 後的結果並不會反映到原始的 yaml 檔案中，這點須特別留意。

### 修改現有的物件(二)：kubectl patch

另外我們也能透過 `kubectl patch` 來直接修改特定的欄位，而不用經過文字編輯器：

**語法**

```bash
kubectl patch <resource_type> <resource_name> -p <json-patch>
```

舉例來說，我想更新 Pod 的 image 為 nginx:1.19.1，可以這樣做：

* 建立一個 Pod：

```bash
kubectl run nginx --image nginx
```

* 使用 kubectl patch 更新 image：

```bash
kubectl patch po nginx -p '{"spec":{"containers":[{"name":"nginx","image":"nginx:1.19.1"}]}}'
```

* 另外，也可以加上 `--dry-run=client -o yaml` 先預覽一下：

```bash
kubectl patch po nginx -p '{"spec":{"containers":[{"name":"nginx","image":"nginx:1.19.1"}]}}' --dry-run=client -o yaml
```


`kubectl patch` 比較常用在腳本、自動化流程等「無法使用文字編輯器」的情況，因為當你手動把 json patch 打出來時，`kubectl edit` 說不定早就改好了。


### 今日小結

今天把過去介紹的 kubectl 指令做了大致的整理，並補充了一些相關的小技巧來加快操作的速度。總之指令這種東西就是熟能生巧，如果真的忘記怎麼使用，-h 與官網會是你的好朋友。

> 想快速去到官網，除了加入書籤外，在瀏覽器輸入「k8s.io」即可。

----
**參考資料**

[kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)




