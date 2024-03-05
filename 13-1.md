# **ALM**: kubectl edit

在上一章中我們使用了兩種方式來更新'Deployment`的`image`:
  1. `kubectl apply`
  2. `kubectl set image`

不過，「更新」並不會僅限於`image`，例如`tolerations`、`container's name`等等。這時候就可以使用`kubectl edit`來進行更新:

```bash
kubectl edit <resource_type>/<resource_name>
```

**範例情境1**

環境由兩個`node`組成:
  * controlplane
  * node01

今天我在`node01`上部署了一個`Deployment`:
```bash
kubectl create deployment nginx --image nginx --replicas 3
```

接著，我想要將`node01`設置成**只有**`nginx`的服務可以運行，因此我設置了`taint`:
```bash
kubectl taint node node01 app=nginx:NoExecute
```

但是，我並沒有幫`Deployment`設置任何的`tolerations`，所以`Pod`會因為`taint`而無法運行。這時候就可以使用`kubectl edit`來進行更新:
```bash
kubectl edit deployment/nginx
```
這時會會進入文字編輯器，我幫`Deployment`加上了`tolerations`:
```yaml
...
...(省略)...
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec: # 注意是在spec.template.spec下，而不是在spec下
      tolerations:
      - key: app
        operator: Equal
        value: nginx
        effect: NoExecute
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
...
...(省略)...
```

儲存離開後，會看到系統提示:
```text
deployment.apps/nginx edited
```

這時候，更新後的`deployment`就會開始運行了:
```bash
controlplane $ kubectl get po -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
nginx-75bf4f68b9-64zz2   1/1     Running   0          3m49s   192.168.1.9   node01   <none>           <none>
nginx-75bf4f68b9-66ct7   1/1     Running   0          3m52s   192.168.1.8   node01   <none>           <none>
nginx-75bf4f68b9-85bg5   1/1     Running   0          3m53s   192.168.1.7   node01   <none>           <none>
```

**範例情境2**

今天我運行了一個`Pod`:
```bash
kubectl run nginx --image nginx
```

接著，我想要將`Pod`的`cpu request`設置為`700m`，這時候就可以使用`kubectl edit`來進行更新:
```bash
kubectl edit po nginx
```
修改成以下:
```yaml
...
...(省略)...
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    resources: 
      requests:
        cpu: "700m"
...
...(省略)...
```

儲存離開後，會看到系統給出提示:
```text
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
# pods "nginx" was not valid:
# * metadata: Invalid value: "BestEffort": Pod QoS is immutable
# * spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
...
...(省略)...
```
重點是這行:
```text
# * spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
```

也就是說，除了以下欄位外，其餘是不允許直接`edit`的:
  * `spec.containers[*].image`
  * `spec.initContainers[*].image`
  * `spec.activeDeadlineSeconds`
  * `spec.tolerations` (只允許新增)
  * `spec.terminationGracePeriodSeconds` (如果之前是負數，則允許設置為1)

不過，雖然直接`edit`行不通，不過我們不儲存離開後，會發先系統提示:
```text
error: pods "nginx" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-3168862298.yaml"
error: Edit cancelled, no valid changes were saved.
```

所以我們剛才`edit`的內容會被儲存到`/tmp/kubectl-edit-3168862298.yaml`，我們可以使用以下方式將這個檔案重新套用:
```bash
kubectl replace -f /tmp/kubectl-edit-3168862298.yaml --force --grace-period=0
```

這會直接刪除掉現有的`Pod`，然後根據`/tmp/kubectl-edit-3168862298.yaml`重新建立一個新的`Pod`:
```bash
$ kubectl get po nginx
NAME                     READY   STATUS    RESTARTS   AGE
nginx                    1/1     Running   0          48s

$ kubectl describe po nginx | grep -i cpu
    cpu:     700m
```
> 不過，`edit`也不會動到原始的`yaml`檔案，所以同樣要小心使用

以上為為`edit`的使用範例，以下為練習題:

## 練習1


