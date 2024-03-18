# Service account

user account vs service account (for human vs for service)

default service account: 存在於每個namespace中，當沒有指定service account時，會使用這個service account
```bash
kubectl describe po <pod-name> -n <namespace>
# 看mount
``` 

看sa名稱: 
```bash
kubectl get po <pod-name> -n <namespace> -o yaml
# 看spec裡面的serviceAccountName
```


但是default service account權限較少，因此我們可以自己建立service account並指定給pod使用

## service account 內容

token: 用來驗證身份的token

 * 首先建立svc
 * 生成token
 * 生成secret來放token
 * 將svc與token連結

# RBAC for service account

```bash
kubectl create sa <service-account-name>
```

```bash
kubectl create role <role-name> --verb=get,watch,list --resource=pods
```

```bash
kubectl create rolebinding <role-binding-name> --role=<role-name> --serviceaccount=<namespace>:<service-account-name>
```

https://blog.miniasp.com/post/2022/08/24/Understanding-Service-Account-in-Kubernetes-through-MicroK8s


# ? 
kubectl create token <svc-name>


