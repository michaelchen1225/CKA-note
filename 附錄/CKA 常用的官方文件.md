# CKA 常用的官方文件

以下分為不同主題，整理出 CKA 考試中常用的官方文件：

* [kubectl cheat sheet](#kubectl-cheat-sheet)
* [Pod 相關](#pod)
* [Deployment 相關](#deployment)
* [configMap & secret 相關](#configmap--secret)
* [Volume 相關](#volume)
* [PV & PVC 相關](#pv--pvc)
* [資源管理](#資源管理)
* [憑證管理](#憑證管理)
* [Ingress](#ingress)
* [Network Policy](#network-policy)
* [權限管理：Security Context、RBAC、Service Account](#權限管理)
* [Cluster upgrade](#cluster-upgrade)
* [etcd 的備份與還原](#etcd-的備份與還原)

### kubectl cheat sheet

目標 | 關鍵字 | 官方文件
-------|--------|-----
kubectl cheat sheet | kubectl cheat sheet | [kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/cheatsheet/)

### Pod 

目標 | 關鍵字 | 官方文件
-------|--------|-----
command and Arguments  | cmd and arg   | [Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
multi container | sidecar | [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
Init Containers | init container | [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

### Deployment 

目標 | 關鍵字 | 文件
-------|--------|-----
rolling update | rolling update | [Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
Rollback | deploy | [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) -> 搜尋「rollback」
標記 CHANGE-CAUSE | deploy | [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) -> 搜尋「cause」

### Sceduling Pod 

目標 | 關鍵字 | 文件
-----|--------|------
nodeName & nodeSelector | nodename、pod to node | [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)
Node affinity | affinity | [Assign Pods to Nodes using Node Affinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)
Taint & Tolerations | taint | [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)


### configMap & secret 

目標 | 關鍵字 | 文件
-----|--------|------
在 Pod 中使用 configMap  | pod use configmap | [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-a-container-environment-variable-with-data-from-a-single-configmap)
在 Pod 中使用 secret | pod use secret| [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)

### Volume 

目標 | 關鍵字 | 文件
-----|--------|------
emptyDir、hostPath、configmap | volume | [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
secret volume | secret | [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#use-case-dotfiles-in-a-secret-volume) --> Use case: dotfiles in a secret volume

### PV & PVC 

目標 | 關鍵字 | 文件
-----|--------|------
在 Pod 中掛在 hostPath PV| pv in pod | [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)
local PV | pv | [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) -> 搜尋 Local

### 資源管理

目標 | 關鍵字 | 文件
-----|--------|------
Pod QoS | qos | [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
LimitRange | limitrange | [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)
Namespace Quota | quota for ns | [Configure Memory and CPU Quotas for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)


### 憑證管理

目標 | 關鍵字 | 文件
-----|--------|------
使用者憑證相關   | csr | [Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/) --> How to issue a certificate for a use
kubeconfig | kubectl config | [kubectl config](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/)

### Ingress 

目標 | 關鍵字 | 文件
-------|--------|-----
Ingress  | ingress   | [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource)

### Network Policy


目標 | 關鍵字 | 文件
-----|--------|------
設定 Network Policy | network policy yaml | [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

### 權限管理

目標 | 關鍵字 | 文件
-----|--------|------
設定 security context | security context | [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
RBAC | rbac | [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
Pod 中的 service account | service account | [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

### Cluster upgrade

目標 | 關鍵字 | 文件
-----|--------|------
升級 kubeadm cluster  | kubeadm upgrade | [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

### etcd 的備份與還原

目標 | 關鍵字 | 文件
-----|--------|------
備份/還原 etcd | etcd snapshot | [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
