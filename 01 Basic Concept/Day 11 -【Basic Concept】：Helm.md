### 今日目標

* 了解 Helm 的用途與架構

* 安裝 Helm

* Helm 的實作
  * 建立 Chart
  * 透過 Chart 部署應用服務
  * 更新 Chart

* 部署他人分享的 Chart

* Helm 的常用指令彙整

今天是「Basic Concept」章節的最後一篇，我們來看一個好用的套件管理工具：Helm。

> 本日內容不屬於 CKA 考試範圍。

### 什麼是 Helm？

Helm 是一項 CNCF 的專案，能將 k8s 中的應用打包起來，方便我們管理、部署、升級。

那為什麼應用需要打包？在實務中，一項 k8s 應用服務會由很多份 yaml 來組成，例如常聽見的微服務架構。而這會造成以下問題：

* 因為有很多的 yaml，所以部署整套應用時就要一直複製貼上、複製貼上，非常麻煩。

* 因為部署環境的不同，可能需要針對應用服務做一些微調。但如果這些微調的 yaml 都散落在不同的地方，就會讓管理變得很困難。

* 當整套應用需要升級時，一堆的 yaml 很難有效率的做到統一的管理與升級。

為了解決這些問題， Helm 將一套應用服務打包起來，形成一個叫做「chart」的檔案，我們就能針對 chart 來部署、管理、調整、升級整套應用服務。另外，chart 也可以透過 repository 來分享，讓其他人可以快速的部署應用服務，也有利於版本控制。

> 以[官網](https://helm.sh/)的話來說，Helm 有以下四大特色：

* Manage Complexity
* Easy Updates
* Simple Sharing
* Rollback

### 安裝 Helm

> 以下以 Debian/Ubuntu 為例，其他作業系統請參考[官網](https://helm.sh/docs/intro/install/)。

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

安裝後查看是否安裝成功：

```bash
helm version
```
輸出：
```text
version.BuildInfo{Version:"v3.15.4", GitCommit:"fa9efb07d9d8debbb4306d72af76a383895aa8c4", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

### Helm 的實作：建立並部署第一個 Chart

我們來實際打包一個簡單的應用服務，並透過 Helm 來部署。

> 該應用服務包含兩個部分：一個是 Deployment，一個是 Service。

需要的檔案已經寫好了，將專案目錄 clone 下來即可：
```bash
git clone https://github.com/michaelchen1225/helm-demo.git
cd ./helm-demo
```

此目錄就是一個可用的 Helm chart，我們先看一下這個 Helm chart 的基本目錄結構：
```text
helm-demo
|-- Chart.yaml
|-- charts
|-- templates
|   |-- nginx-deploy.yaml
|   |-- nginx-deploy-svc.yaml
|   |-- NOTES.txt
|-- values.yaml
```
* **Chart.yaml**：用來描述該 chart 的基本資訊，例如 chart 的名稱、版本、描述等。
* **charts**：放置「子 chart」的目錄，通常有相依性的 chart 會放在這裡，這個目錄預設是空的。
* **templates**: 放置 k8s 資源 yaml 的目錄，以下為我們自行添加的 yaml：
  * nginx-deploy.yaml：Deployment 的 yaml
  * nginx-deploy-svc.yaml：Service 的 yaml
  * NOTES.txt：這個檔案會在部署 chart 時顯示，用來提醒使用者一些重要資訊。
* **values.yaml**：用來設定一些參數，這些參數會被套用在 templates 中使用，例如：image 的版本、replicas 的數量等。當我們需要對不同部署環境微調應用服務時，修改這個檔案就可以了。

> 上述為一個 helm chart 必須的基本專案結構，該結構可以用 `helm create` 來初始化，例如：
```bash
helm create helm-demo-2
```

有了一個基本的 Helm chart 結構後，我們先來撰寫 Chart.yaml：
```yaml
apiVersion: v2
name: helm-demo
description: helm demo from day 11
type: application
version: 0.1.0
appVersion: "1.0.0"
```
* **apiVersion**： Helm 的適用版本，目前是 v2。
* **name**：chart 的名稱。
* **description**：chart 的描述。
* **type**：chart 的類型，可以是 application、library，application 代表這是一個 k8s 應用服務，而 Libary 則類似於一個函式庫，可以被其他 chart 引用。
* **version**：chart 的版本。
* **appVersion**：應用服務的版本。

接著，我們來看一下 nginx-deploy.yaml 的內容：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: nginx:1.15
        name: nginx
```

為了讓這個 yaml 能被彈性的應用，我們能將一些寫死的值改成以下格式：
```text
{{ .Object.Parameter }}
```
這個「.Object」可以是以下幾種：
* .Chart：Chart.yaml 的內容
* .Release：每次部署 chart 都會有帶有一個 release 物件，包含了這次部署的相關資訊。
* .Values：values.yaml 的內容

這樣說有點抽象，這裡直接實際把 nginx-deploy.yaml 修改成底下這樣：
```bash
vim templates/nginx-deploy.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: {{ .Release.Name }}-nginx-deploy
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        name: nginx
```

> 這樣一來，這個 Deployment 在部署實會被彈性調整的地方有：
* metadata.name：名稱會加上 .Release 物件中的 Name 做前綴。
* spec.replicas：副本數量會使用「values.yaml」中 replicas 的值。
* spec.containers.image：image 會使用「values.yaml」中 image 的 repository 與 tag。

實際的 values.yaml 的內容長這樣：
```yaml
replicas: 3

image:
  repository: nginx
  tag: 1.15
```

設定好後，我們來檢查一下這個 chart 的設定是否正確：
```bash
# 要在 helm-demo 目錄下執行
helm lint .
```
輸出：
```text
==> Linting .
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

這樣就代表這個 chart 沒有問題，如果有誤你可能會看到一些錯誤訊息，例如：
```text
==> Linting .
[INFO] Chart.yaml: icon is recommended
[ERROR] templates/nginx-deploy.yaml: unable to parse YAML: error converting YAML to JSON: yaml: line 19: mapping values are not allowed in this context

Error: 1 chart(s) linted, 1 chart(s) failed
```

接著可以檢查看看將「value.yaml」的值帶入後的 template yaml 是否符合預期：
```bash
helm template .
```

一切 OK 後，離開 helm-demo 目錄，我們就可以透過 Helm 部署這個 chart：
```bash
cd ~
helm install first-chart ./helm-demo
```
輸出：
```text
LAST DEPLOYED: Thu Aug 29 10:24:04 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Hi !
This is the helm demo from Day 11
```
> 這會建立一個名為 first-chart 的 release，並部署 helm-demo 這個 chart 在 default namespace 中。

檢查一下部署狀態：
```bash
kubectl get deploy,svc
```
```text
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/first-chart-nginx-deploy   3/3     3            3           90s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes         ClusterIP   10.96.0.1      <none>        443/TCP   27d
service/nginx-deploy-svc   ClusterIP   10.96.111.48   <none>        80/TCP    90s
```

查看一下目前安裝過的 release：
```bash
helm list
```
輸出：
```text
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
first-chart     default         1               2024-08-29 10:24:04.562088695 +0000 UTC deployed        helm-demo-0.1.0 1.0.0 
```

這樣就完成了一個簡單的 Helm chart 打包與部署，接著嘗試更新這個 chart，把 replicas 數量改成 1：

* 修改 ./helm-demo/values.yaml：
```yaml
replicas: 1
...(略)
```

* 更新 release：
```bash
helm upgrade first-chart ./helm-demo
```
輸出：
```text
Release "first-chart" has been upgraded. Happy Helming!
NAME: first-chart
LAST DEPLOYED: Thu Aug 29 10:25:15 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
Hi !
This is the helm demo from Day 11
```
> REVISON 2 代表這是第二次更新。

* 檢查更新後的 replicas 數量：
```bash
kubectl get deploy first-chart-nginx-deploy
```
```text
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
first-chart-nginx-deploy   1/1     1            1           2m8s
```

* 也可以快速 rollback 到之前的版本，語法為 `helm rollback <release-name> <revision>`：
```bash
helm rollback first-chart 1
```
輸出：
```text
Rollback was a success! Happy Helming!
```

檢查後可以發現 replicas 數量又變回 3 了。我們可以用 `helm history <release-name>` 來查看 release 的歷史紀錄：
```bash
helm history first-chart
```
輸出：
```text
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Thu Aug 29 10:24:04 2024        superseded      helm-demo-0.1.0 1.0.0           Install complete
2               Thu Aug 29 10:25:15 2024        superseded      helm-demo-0.1.0 1.0.0           Upgrade complete
3               Thu Aug 29 10:27:19 2024        deployed        helm-demo-0.1.0 1.0.0           Rollback to 1
```

如果想比對兩個 REVISON 的差異，可以使用 `helm diff revision <release-name> <revision-1> <revision-2>`： 
```bash
helm diff revision first-chart 2 3
```

最後我們將這個 release 刪除：
```bash
helm uninstall first-chart
```

### 修改 Helm 的 value 設定

除了直接修改 values.yaml 之外，我們也可以透過 `--set` 來修改 Helm 的 value 設定：

* 例如將 nginx 的 image 版本改成 1.16：
```bash
helm install first-chart ./helm-demo --set image.tag=1.16
```

* 驗證一下：
```bash
kubectl get deploy first-chart-nginx-deploy -o jsonpath='{.spec.template.spec.containers[0].image}'
```
輸出：
```text
nginx:1.16
```

也可以使用 `--values` 來指定一個 yaml 檔案來取代原本的 values.yaml，這樣就能更方便的管理不同環境的設定：

```bash
helm install new-value-chart ./helm-demo --values <path-to-other-values.yaml>
```

### 打包 Chart 並分享

當我們完成一個 chart 的開發後，可以將這個 chart 打包成一個 .tgz 檔案，這樣就能夠分享給其他人使用。

```bash
helm package ./helm-demo
```
輸出：
```text
Successfully packaged chart and saved it to: /root/helm-demo-0.1.0.tgz
```

### 建立 helm chart repository：以 Gitlab 為例

一個標準的 helm chart repository 應該長這樣：

```text
charts/
  |
  |- index.yaml
  |
  |- my-chart.tgz
```

重點是那個 index.yaml，這個檔案會記錄這個 repository 中所有的 chart 資訊，例如：chart 的名稱、版本、描述等。

以下為建立一個 helm chart repository 的步驟：

* 建立一個新目錄，例如：`helm-repo`：

```bash
mkdir helm-repo
```

* 進入 helm-repo 後先打包 chart：

```bash
helm package /root/helm-demo
```
```text
Successfully packaged chart and saved it to: /root/helm-repo/helm-demo-0.1.0.tgz
```

* 生成 index.yaml：

```bash
helm repo index 
```

* 新增 .gitlab-ci.yml，讓 Gitlab CI 在 push 後自動將 chart 丟上去：

```yaml
stages:
  - upload

upload:
  image: curlimages/curl:latest
  stage: upload
  script:
    - CHART=$(ls | grep .gz)
    - echo $CHART
    - 'curl --fail-with-body --request POST --user gitlab-ci-token:$CI_JOB_TOKEN --form "chart=@helm-demo-0.1.0.tgz" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/helm/api/stable/charts"'
```

* 提交變更到 Gitlab、等待 CI 完成後，前往左邊選單中的 `Deploy -> Package registry` 查看丟上去的 helm package：

  ![alt text](image-2.png)


* 將這個 repository 加入 helm repo list 當中：

```bash
helm repo add  my-helm-repo https://gitlab.com/api/v4/projects/<project-id>/packages/helm/stable
```
> gitlab 的 project-id 可以在 repo 首頁的右上角找到：
![alt text](image-1.png)

* 加入後 update 一下：

```bash
helm repo update
```

* 查看是否能成功從自己的 repository 搜尋到 helm-demo：

```bash
helm search repo helm-demo
```
```text
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
my-repo/helm-demo       0.1.0           1.0.0           helm demo from day 11
```

* 安裝 helm-demo：

```bash
helm install my-release my-repo/helm-demo -n test-helm-demo --create-namespace
```

* 查看 release：

```bash
helm list -n test-helm-demo
```

* 確認部署是否成功：

```bash
kubectl get all -n test-helm-demo
```

**注意**

* 未來如果有在 repo 中加入新的 chart，記得要重新下 `helm repo index` 來更新 index.yaml 中的資訊！

* 若有修改原本的 chart，記得也要一起修改 Chart.yaml 中的 version 再使用 `helm package` 重新打包。



### 部署他人分享的 Chart

Helm 的 chart 也有類似 Github 的平台來分享，來源分為以下兩者：

1. Artifact Hub
2. 自行加入的 repository

**安裝 Artifact Hub 上的 Chart**

* 前往 [Artifact Hub](https://artifacthub.io/) 搜尋你想要的 Chart，例如：`nginx`。點進去後可以看到這個 Chart 的相關資訊，包含了安裝方式、相依性等。

**安裝自行加入的 repository**

* 使用以下指令加入 repository：
```bash
helm repo add <repo-name> <repo-url>
```

* 加入後，就可以從這個 repository 安裝 Chart：
```bash
helm install <release-name> <repo-name>/<chart-name>
```

* 如果想要更新 repository 的資訊，可以使用：
```bash
helm repo update
```

* 如果想要查看目前加入的 repository，可以使用：
```bash
helm repo list
```

* 如果想要移除 repository，可以使用：
```bash
helm repo remove <repo-name>
```

> 有關其他的 helm 指令用法，可以用善用 `-h` 來查看。

### Helm 的基本指令彙整


以下將 helm 的常用指令彙整，方便日後查詢：

> **注意**：不同 namespace 的操作請用 `-n` 指定，以下預設為 default namespace。

初始化一個 Chart，生成基本的目錄結構：
```bash
helm create <chart-name>
```

檢查 chart 的配置是否正確：
```bash
helm lint <chart-path>
```

查看 chart 的 template 是否符合預期：
```bash
helm template <chart-path>
```

安裝 Chart：
```bash
helm install <release-name> <chart-name>
```

從特定 repo 安裝 Chart：
```bash
helm install <release-name> <repo-name>/<chart-name>
```

安裝 Chart 並修改 value：
```bash
helm install <release-name> <chart-name> --set <key>=<value>
```

安裝 Chart 並帶入新的 value.yaml：
```bash
helm install <release-name> <chart-name> --values <path-to-other-values.yaml>
```

把 Chart 安裝在一個全新的 namespace：
```bash
helm install <release-name> <chart-name> -n <new-namespace> ----create-namespace
```

列出某個 namespace 中的 release：
```bash
helm list -n <namespace>
```

列出所有 namespace 中的 release：
```bash
helm list --all-namespaces
```

解除安裝 Chart：
```bash
helm uninstall <release-name>
```

列出已安裝的 release：
```bash
helm list
```

更新一個 release：
```bash
helm upgrade <release-name> <chart-name>
```

更新一個 release，且 chart 來源為遠端 repo：
```bash
helm repo update
helm upgrade <release-name> <repo-name>/<chart-name>
```

查看某個 release 的版本紀錄：
```bash
helm history <release-name>
```

Rollback 一個 release 到指定 REVISION：
```bash
helm rollback <release-name> <revision>
```

檢查兩次 REVISION 的差異：
```bash
helm diff revision <release-name> <revision-1> <revision-2>
```

打包 Chart 成一個 tgz 檔：
```bash
helm package <chart-name>
```

加入新的 repo：
```bash
helm repo add <repo-name> <repo-url>
```

更新 repo：
```bash
helm repo udpate
```

列出目前可用的 repo：
```bash
helm repo list
```

移除 repo：
```bash
helm repo remove <repo-name>
```

### 今日小結

今天是「Basic Concept」章節的最後一篇，前面我們已經掌握了 k8s 中基本的概念以及相關操作，而隨著面對的 yaml 越來越多，今天也介紹了 Helm 這個方便的套件管理工具，能夠讓我們更有效率的管理、部署、升級應用服務。另外，Helm 與 k8s 一樣，都有相當完整的官方文件可以參考，通常搜尋關鍵字都能找到相關資訊。

明天就會進入新的章節：「Storage」，將會介紹 k8s 中的儲存相關概念，例如 configMap、secret、volume 等，讓我們更了解如何在 k8s 中有效的管理與保存資料。

---

**參考資料**

* [Helm 官網](https://helm.sh/)

* [Installing Helm](https://helm.sh/docs/intro/install/)

* [Artifact Hub](https://artifacthub.io/)

* [Helm Charts Tutorial: A Simple Guide for Beginners](https://devopscube.com/create-helm-chart/)

* [Get started with GitLab's Helm Package Registry](https://about.gitlab.com/blog/2021/10/18/improve-cd-workflows-helm-chart-registry/)

