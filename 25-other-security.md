# image security

使用secret來訪問私有的image registry

```bash
kubectl create secret docker-registry <secret-name> --docker-server=<registry-server> --docker-username=<user> --docker-password=<password> --docker-email=<email>
```

將secret指定給pod使用: spec.imagePullSecrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
    containers:
    - name: private-reg-container
        image: <private-registry>/<image-name>
    imagePullSecrets:
    - name: <secret-name>
```

# security context
https://ithelp.ithome.com.tw/articles/10241665
https://www.qikqiak.com/k8strain/security/security-context/
https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

> linux capabilities (只作用於container層級，不能在pod層級設定)

> container level security context replace pod level security context

(如果container level沒有設定才會使用pod level的設定)


# network policy 自己一篇，到網路篇再講

egress: 只有運許的規則才可以出去(自動允許回復)，子規則: to
ingress: 只有運許的規則才可以進來(自動允許回復)，子規則: from

> 使用label來選擇policy的對象


規則: 彼此之間使用OR，同規則下使用AND

- podSelector: 所有namespace的只要符合podSelector的pod都會被運用
- namespaceSelector: 所有符合namespaceSelector的namespace都會被運用
combo: podSelector + namespaceSelector
> 只有在這個namespace且符合podSelector的pod才會被運用
- ipBlock
- ports

https://kubernetes.io/docs/concepts/services-networking/network-policies/

allow all ingress and egress traffic
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```
