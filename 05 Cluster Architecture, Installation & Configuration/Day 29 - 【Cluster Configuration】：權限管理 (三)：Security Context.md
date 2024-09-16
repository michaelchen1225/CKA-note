### 今日目標

---

* 了解什麼是 Security Context 

* 實作：
  * 設定 UID、GID
  * 設定 fsGroup 來完成檔案權限的共享
  * 設定 Linux Capabilities

***

在前兩天的文章中，我們了解了「人」與「服務」的權限設定，今天我們來看看「**容器內部**」的權限。

容器內部的權限和「Linux 的權限設定」比較有關，例如 UID、GID、Linux Capabilities 等等。當我們在 Pod 中執行特定的指令時，這些指令**預設**會以 root 的身分在容器中執行：

* 建立一個 busybox Pod，執行 `sleep 3600` 指令：
```bash
kubectl run busybox --image=busybox --command -- sleep 3600
```

* 等容器建立後，可以看到 sleep 3600 是以 root 的身分在執行的：
```bash
kubectl exec busybox -- ps aux
```

```text
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 3600
    6 root      0:00 ps aux
```

不過預設用 root 來跑指令，在安全性上是有風險的，因為 root 身分的在容器內可以進行任何操作，包括讀取、寫入、刪除檔案等等。

除了安全上的風險，容器中的「檔案權限」也是個問題。如果在容器中用 root 身分建立了一個檔案，就只有 root 身分可以讀取，其他使用者想讀取出非改權限，會造成一定程度的麻煩。

為了解決上面問題，我們來看看今天的主角：**Security Context**。

## Security Context

Security Context 可以在 yaml 中對 Pod 進行以下設定：

1. **UID、GID**：指定 Pod 中執行的使用者身分、群組身分

2. **SELinux**：指定 SELinux 的標籤

3. **fsGroup**：指定 volume 檔案的群組身分 (GID)

4. **Linux Capabilities**：指定可以有哪些 Linux Capabilities，capabilities 的清單可以參考[這裡](https://man7.org/linux/man-pages/man7/capabilities.7.html)

5. **Seccomp**：過濾 container 中的 system call

6. **allowPrivilegeEscalation**：允許某個 process 的權限大於其 parent process

> Security Context 還有很奪其他玩法，可以參考[官方文件](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.31/#securitycontext-v1-core)

下面我們來實際嘗試 Security Context 三種設定：UID & GID、fsGroup、Linux Capabilities。

### 設定 UID、GID

在 Linux 中，每個使用者都會有自己的 UID、GID，這會直接影響到使用者的權限。

UID、GID 可以用 `id` 指令查看：

```bash
id
```
輸出：
```text
id
uid=0(root) gid=0(root) groups=0(root)
```

* 我們可以在 Pod yaml 的 `spec.securityContext` 中指定容器「預設」的 UID、GID：

```yaml
# sc-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-test
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 2000
  containers:
  - name: sc-test
    image: busybox
    command: [ "sleep", "3600"]
```
```bash
kubectl apply -f busybox.yaml
```

* 查看 sc-test 中的 process，可以看到 sleep 3600 是以 UID=1000 的身分在執行:

```bash
kubectl exec sc-test -- ps aux
```
```text
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 3600
    7 1000      0:00 ps aux
```

* 查看 sc-test 中的使用者身分，是否為 uid=1000 gid=2000：
  
```bash
kubectl exec -it sc-test -- id
```
```text
uid=1000 gid=2000 groups=2000
```

如果 Pod 中有多個容器，某個容器不想使用預設的 UID、GID，可以在`spec.containers[].securityContext` 中指定：

```yaml
# multi-sc-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-sc-test
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 2000
  containers:
  - name: use-default
    image: busybox
    command: [ "sleep", "3600"]
  - name: use-custom
    image: busybox
    command: [ "sleep", "3600"]
    securityContext:
      runAsUser: 1001
      runAsGroup: 2001
```
```bash
kubectl apply -f multi-sc-test.yaml
```

* 查看使用「預設 UID、GID 」的容器：

```bash
kubectl exec multi-sc-test -c use-default -- id
```
```text
uid=1000 gid=2000 groups=2000
```

* 看看使用「自訂 UID、GID」的容器：

```bash
kubectl exec multi-sc-test -c use-custom -- id
```
```text
uid=1001 gid=2001 groups=2001
```

> 所以，Pod Level 的 Security Context 用來設定所有容器的預設值，如果容器有自己的 Security Context，則會覆蓋 Pod Level 的設定。

### fsGroup

為了解決檔案權限的問題，Security Context 可以讓容器裡「volume 的檔案」都屬於同一個**群組**，進而達到共享的目的，這可以使用 `spec.securityContext.fsGroup` 來設定：

> 下面的 fsgroup-demo 有兩個容器：writer & reader。

```yaml
# fsgroup-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fsgroup-demo
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "sleep 1h"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "sleep 1h"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
    securityContext:
      runAsUser: 1000
      runAsGroup: 2000
  volumes:
  - name: shared-data
    emptyDir: {}
```
```bash
kubectl apply -f fsgroup-demo.yaml
```

* Pod 跑起來後，我們先開啟一個終端，進入 writer 容器：

```bash
kubectl exec -it fsgroup-demo -c writer -- sh
```

* 先查看一下 writer 容器的身分：
```bash
id
```
```text
uid=0(root) gid=0(root) groups=0(root),10(wheel),2000
```

* 然後在 /data 目錄下建立一個 test.txt 檔案：

```bash
echo "test fsgroup" > /data/test.txt
```

* exit 離開 writer 容器後，我們使用 reader 容器來查看 /data 目錄下的檔案與權限：

```bash
kubectl exec  fsgroup-demo -c reader -- ls -l /data
```
```text
total 4
-rw-r--r--    1 root     2000            13 Sep 16 05:21 test.txt
```

可以看到，雖然 writer 是 root 的身分，但是建立出的 test.txt 檔案只有擁有者為 root，而所屬群組則是 2000，而 2000 群組有「r」權限，所以 reader 容器可以讀取 test.txt 檔案：

```bash
kubectl exec  fsgroup-demo -c reader -- cat /data/test.txt
```
輸出：
```text
test fsgroup
```


### 設定 Linux Capabilities

Linux Capabilities 是一種可以讓「process 擁有部分 root 權限」的機制，可以讓 process 在不需要完整 root 權限的情況下執行某些操作。

* 建立一個沒有任何 capabilities 的 Pod：

```yaml
# normal.yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sh", "-c", "sleep 1h"]
```
```bash
kubectl apply -f normal.yaml
```

* 查看 normal process 擁有的 capabilities：

```bash
kubectl exec  normal -- cat /proc/1/status | grep CapPrm
```
```text
CapPrm: 00000000a80425fb
```
> 以16進位表示的 capabilities，每個 bit 代表一個 capability (等下比較時我們會轉成2進位)

* 創建另一個 Pod，加上 `SYS_TIME` 的 capability：

```yaml
# set-time.yaml
apiVersion: v1
kind: Pod
metadata:
  name: set-time
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sh", "-c", "sleep 1h"]
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
```
```bash
kubectl apply -f set-time.yaml
```

* 查看 set-time process 擁有的 capabilities：

```bash
kubectl exec -it set-time -- cat /proc/1/status | grep CapPrm
```
```text
CapPrm: 00000000aa0425fb
```

底下我們將兩個 Pod 的 capabilities 轉成2進位，這樣比對起來比較清楚：
  
**normal**：

```text
1010 1000 0000 0100 0010 0101 1111 1011
```

**set-time**：

```text
1010 1010 0000 0100 0010 0101 1111 1011
```

可以看到 set-time 多了一個 bit (左邊數來第七個)，這個 bit 就是 SYS_TIME 的 capability。


### 環境清理

> 練習後別忘了清理環境喔

* 清除 default namespace 中的所有的 Pod：

```bash
kubectl delete pod --all
```

### 今日小結

Security Context 可以讓我們控制 Pod 的權限、檔案權限、capabilities 等等，關於其他 Security Context 的玩法，可以參考[官方文件](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)。

今天是 【Cluster Configuration】的最後一篇，明天我們來看這個系列的最後一個章節：【Troubleshooting & Monitoring】。

-----
**參考資料**

[Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

[Security Context](https://www.qikqiak.com/k8strain/security/security-context/)



