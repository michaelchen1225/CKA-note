# Backup and Restore

除了定期升級外，為避免突發情況造成資料遺失「備份」也是一個相當重要的工作。養成定期備份的習慣，可以讓你在資料遺失快速的透過「還原」來恢拯救系統。

在`k8s cluster`中，需要重點備份的資料有:
  * 資源設定檔
  * `etcd`

## 資源設定檔

在過去的操作中，我們不外乎透過以下兩種方式來建立資源:
  * **指令**: 例如`kubectl create`、`kubectl run`
  * **yaml檔案**: 透過`kubectl apply -f`或`kubectl create -f`來建立

在[Day 04](04-1.md)中曾經提過，使用`yaml`檔雖然速度不及直接使用指令，但是在做「版控」時會輕鬆許多，畢竟我們很難掌握到底是誰、什麼時候、用什麼指令建立的資源。透過版控系統，例如`git`，可以讓我們清楚的知道每一次的變更，也能在資遺失時快速的復原。

但是有時我們仍會使用指令來建立資源，在備份時，我們可以透過以下方式將資源設定檔備份下來:
```bash
kubectl get all --all-namespaces -o yaml > all-resources.yaml
```

> 需要注意的是，這樣的備份方式較為簡略，並不能做到「全面」的備份。

## etcd

要做到較為全面的備份，就必須備份`etcd`。`etcd`是`k8s`的資料庫，使用`key-value`的方式來儲存`k8s`的重要資料和關鍵訊息，例如現有資源、`cluster`狀態等等。

備份`etcd`的方式有很多種，這裡介紹一種簡單的方式:`snapshot`。`snapshot`是`etcd`內建的一個備份機制，透過`etcd`的「command line client」---`etcdctl`，我們可以透過`snapshot(快照)`將`etcd`的資料備份下來，並在日後還原`etcd`。

備份與還原的基本流程圖如下:
![etcd-snapshot](20-1-etcd-backup.png)

## 實際備份etcd

為了實測備份後還原的效果，這裡先建立一個`deployment`:
```bash
kubectl create deployment nginx --image nginx --replicas 3
```

將需要的環境變數設定好:
```bash
export ETCDCTL_API=3
```

建立`etcdctl snapshot`的指令格式如下:
```bash
 etcdctl --endpoints=<listen-client-urls> \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```

```bash
$ cat /etc/kubernetes/manifests/etcd.yaml | grep listen-client
    - --listen-client-urls=https://127.0.0.1:2379,https://172.30.1.2:2379 # 需要這行
$ cat /etc/kubernetes/manifests/etcd.yaml | grep file         
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt # 需要這行
    - --key-file=/etc/kubernetes/pki/etcd/server.key # 需要這行
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt # 需要這行
```
我們所需的資料有:
  * 127.0.0.1:2379 (--listen-client-urls)
  * /etc/kubernetes/pki/etcd/server.crt (--cert-file)
  * /etc/kubernetes/pki/etcd/server.key (--key-file)
  * /etc/kubernetes/pki/etcd/ca.crt (--trusted-ca-file)

> 這些資料所代表的意涵將會在後續章節中說明。

透過`etcdctl`備份`etcd`，並將`snapshot`儲存到/opt/etcd-backup.db:
```bash
etcdctl --endpoints=127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

接著我們刪掉`nginx`的`deployment`:
```bash
kubectl delete deploy nginx
```

還原`etcd`的`snapshot`c還原到一個新的目錄`/var/lib/etcd-restore`:
```bash
etcdctl snapshot restore --data-dir /var/lib/etcd-restore /opt/etcd-backup.db
```

由於`etcd`資料儲存的路徑已經移到了`/var/lib/etcd-restore`，所以我們需要修改`etcd`的`manifest`檔案:
```bash
$ vim /etc/kubernetes/manifests/etcd.yaml
```

修改如下:

```yaml
...
...(省略)
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-restore # 改這裡(原本這裡是/var/lib/etcd)
      type: DirectoryOrCreate
    name: etcd-data 
status: {}
```

> 這裡修改的意義將會在後續的章節中說明。


儲存後離開，系統會自動重啟`etcd`(因為是`static pod`)，等待一段時間後，我們可以看到`nginx`的`deployment`又回來了:
```bash
$ kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           10m
```

**補充**

在等待`etcd`重啟的過程中，我們可以透過以下指令來確認`etcd`的狀態:
```bash
$ crictl ps -a
```
or
```bash
$ docker ps -a
```

`etcd`成功會來的畫面如下:
```bash
$ crictl ps -a
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
6caf9a0d570db       a0eed15eed449       40 seconds ago      Running             etcd                      0                   55b3a0838bc6d       etcd-controlplane
10af7b38f57ad       7ace497ddb8e8       2 minutes ago       Running             kube-scheduler            1                   3763f767b40cb       kube-scheduler-controlplane
ce5f8247e3043       0824682bcdc8e       2 minutes ago       Running             kube-controller-manager   1                   a26
# 這裡的etcd的建立時間是40秒前，明顯比其他的pod晚很多，並且是running的狀態
```

如果`etcd`不斷地重啟失敗，就代表你可能上面有某部分的操作失敗了，建議檢察一下。

