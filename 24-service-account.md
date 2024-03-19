# Service account

前面介紹的各種驗證與授權的機制，我們主要聚焦在「人」的身分上，也就是`user account`。除了「人」之外，「服務」是另外一種會與k8s互動的對象，因此當然也需要驗證與授權的機制。而這個驗證機制就是今天的主題: `service account`。

## What is service account

在k8s中，可分為兩種使用者: 

  1. 人: 也就是一般使用者，使用`user account`來驗證身分(例如前一章的jane)

  2. 服務: 例如一個pod，使用`service account`來驗證身分

總而言之，只要「不是人」就用`service account`。

## default service account

你可能會說: 
> 之前建立pod時，並沒有提到要指定service account啊?

這是因為每個namespace都有一個`default service account`，當你沒有指定service account時，k8s會自動綁定這個`default service account`到pod上。

我們來觀察一下這個`default service account`:

```bash
$ kubectl get sa               
NAME      SECRETS   AGE
default   0         15d
```

假設我今天建立了一個新的namespace，k8s會自動建立一個`default service account`在這個namespace中:

```bash
$ kubectl create ns new-ns
$ kubectl get sa -n new-ns
NAME      SECRETS   AGE
default   0         0s
```

當我們在new-ns中建立一個pod時，k8s會自動綁定`default service account`到這個pod上:

```bash
$ kubectl run nginx --image=nginx -n new-ns
$ kubectl get po nginx -n new-ns -o yaml | grep serviceAccountName
    serviceAccountName: default
```

另外在輸出中的`spec.containers.volumeMounts`中，也可以看到`default service account`被掛載到了pod中:

```bash
$ kubectl get po nginx -n new-ns -o yaml | grep -A 3 volumeMounts
  volumeMounts:
  - mountPath: /var/run/secrets/kubernetes.io/serviceaccount # 掛載點
    name: kube-api-access-sctkc
    readOnly: true
```

這個pod中的掛載點`/var/run/secrets/kubernetes.io/serviceaccount`，是用來存放`default service account`的token的地方:
```bash
$ kubectl exec -it nginx -n new-ns -- ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt  namespace  token
```  

## service account with RBAC

不過為了安全起見，這個`default service account`的權限是較少的，因此我們可以自己建立`service account`並指定給pod使用，再透過`RBAC`來控制這個`service account`的權限。

我們來看一個範例:

* 建立service account
```bash
kubectl create sa read-pod-sa -n new-ns
```

* 建立role
```bash
kubectl create role pod-reader --verb=get,watch,list --resource=pods -n new-ns
```

* 建立role binding
```bash
kubectl create rolebinding pod-reader-binding --role=pod-reader --serviceaccount=new-ns:read-pod-sa -n new-ns
```

驗證一下:
```bash
$ kubectl auth can-i get pods -n new-ns --as system:serviceaccount:new-ns:read-pod-sa 
yes
```

## 在pod中使用service account

在yaml中的`spec.serviceAccountName`中指定:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: new-ns
spec:
  containers:
  - image: nginx
    name: nginx
  serviceAccountName: read-pod-sa # 指定service account
```

我們進入pod裡面使用curl測試一下:
  
```bash
kubectl exec -it nginx -n new-ns -- /bin/sh
```
```bash
CACERT='/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc.cluster.local:443/api/v1/namespaces/new-ns/pods/
```

如果成功會看到new-ns中的pod列表:
```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "4450"
  },
  "items": [
    {
      "metadata": {
        "name": "nginx",
        "namespace": "new-ns",
        "uid": "cc97eabb-bd4d-4173-ba9b-0190dd3b54fc",
        "resourceVersion": "3946",
        "creationTimestamp": "2024-03-19T04:55:59Z",
        "labels": {
          "run": "nginx"
        },
        "annotations": {
...
...(省略)
...
```

> 如果失敗，可以用`kubectl auth can-i`檢查一下RBAC是否設定正確

# Ref
[service account](https://kubernetes.io/docs/concepts/security/service-accounts/)
[service account in pod](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
[透過 MicroK8s 認識 Kubernetes 的 Service Account (服務帳戶)](https://blog.miniasp.com/post/2022/08/24/Understanding-Service-Account-in-Kubernetes-through-MicroK8s)

