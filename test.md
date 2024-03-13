# **TLS in cluster**
## **Client**
### ***管理員***
#### admin.crt
#### admin.key
### ***kube-scheduler***
#### scheduler.crt
#### scheduler.key
### ***kube-controller-manager***
#### controller-manager.crt
#### controller-manager.key
### ***kubelet***
#### kubelet.crt
#### kubelet.key
### ***kube-proxy***
#### proxy.crt
#### proxy.key
### ***kube-apiserver***
#### **to etcd**
##### apiserver-etcd-client.crt
##### apiserver-etcd-client.key
#### **to kubelet**
##### apiserver-kubelet-client.crt
##### apiserver-kubelet-client.key

## **Server**
### ***kube-apiserver***
#### apiserver.crt
#### apiserver.key
### ***etcd***
#### etcd.crt
#### etcd.key
### ***kubelet***
#### kubelet.crt
#### kubelet.key

## **CA**
### ca.crt
### ca.key


