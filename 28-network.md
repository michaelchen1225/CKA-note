# network CNI

* 查看cni:
```bash
kubectl get pods -n kube-system
```

> 例如: `calico`、`flannel`、`cilium`、`weave`等

* 通常支援的cni執行檔會在`/opt/cni/bin/`目錄
```bash
ls /opt/cni/bin/
```
輸出:
```text
andwidth  calico       dhcp   firewall  host-device  install  loopback  portmap  sbr     tap     vlan
bridge     calico-ipam  dummy  flannel   host-local   ipvlan   macvlan   ptp      static  tuning  vrf
```

* 通常正在使用的`cni`設定檔會在`/etc/cni/net.d/`目錄
```bash
ls /etc/cni/net.d/
```
輸出:
```text
10-canal.conflist  calico-kubeconfig
```

來看一下設定檔:
```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico", # 使用calico
      "log_level": "info",
      "log_file_path": "/var/log/calico/cni/cni.log",
      "datastore_type": "kubernetes",
      "nodename": "controlplane",
      "mtu": 0,
      "ipam": {
          "type": "host-local",
          "subnet": "usePodCidr"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    },
    {
      "type": "bandwidth",
...
...(省略)
```

為了串聯整個cluster的node，每個node上都會有一個cni的agent，這個agent會負責處理pod之間的網路通訊:
```bash
kubectl get daemonset -n kube-system
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
canal        2         2         2       2            2           kubernetes.io/os=linux   18d
kube-proxy   2         2         2       2            2           kubernetes.io/os=linux   18d
```


## service 

CNI 和 Service 在 Kubernetes 中各自扮演不同的角色：

CNI 負責提供容器網路的基礎設施，包括 IP 地址分配和路由設定。
Service 提供了一種穩定的方式來訪問 Pod，使得其他 Pod 或外部客戶端可以穩定地訪問一組 Pod。

service type recap:
[Day 6](06-1-svc.md)

* ClusterIP: 只能在cluster內部存取

* NodePort: 在每個cluster的node上開一個port，可以透過node的ip:port存取

## IP範圍

* pod的IP範圍由cni決定:
```bash
cat /etc/cni/net.d/10-canal.conflist | grep -A 3 -i ipam
```
通常長這樣:
```
"ipam":{"ranges":[[{"subnet":"10.5.0.0/24"}]],"type":"host-local"}
```

但也有可能是這樣:
```text
      "ipam": {
          "type": "host-local",
          "subnet": "usePodCidr" 
      },
```

或者，也可以直接查看cni的agent(pod):
```bash
k get po -n kube-system weave-net-wbdzb -o yaml | grep  -i range -A 2
    - name: IPALLOC_RANGE
      value: 10.244.0.0/16
```

如果長上面那樣，代表cni使用的IP range 取決於k8s的設定，可以透過以下指令查看:
```bash
$ k get node -o yaml | grep -i podcidr
    podCIDR: 192.168.0.0/24
    podCIDRs:
    podCIDR: 192.168.1.0/24
    podCIDRs:
```

* service的IP範圍由kube-apiserver決定:
```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -i range 
 - --service-cluster-ip-range=10.96.0.0/12
```

範例: 建立了一個pod叫做nginx，並幫他建立一個service
```bashcontrolplane $ iptables -L -t nat | grep nginx
KUBE-MARK-MASQ  all  --  anywhere             anywhere             /* masquerade traffic for default/nginx external destinations */
KUBE-EXT-2CMXP7HKUVJN7L6M  tcp  --  anywhere             anywhere             /* default/nginx */ tcp dpt:30608
KUBE-MARK-MASQ  all  --  192.168.1.4          anywhere             /* default/nginx */
DNAT       tcp  --  anywhere             anywhere             /* default/nginx */ tcp to:192.168.1.4:80
KUBE-SVC-2CMXP7HKUVJN7L6M  tcp  --  anywhere             10.99.252.88         /* default/nginx cluster IP */ tcp dpt:http
KUBE-MARK-MASQ  tcp  -- !192.168.0.0/16       10.99.252.88         /* default/nginx cluster IP */ tcp dpt:http
KUBE-SEP-LJUUEGC24UMYBEWU  all  --  anywhere             anywhere             /* default/nginx -> 192.168.1.4:80 */
controlplane $ k get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        18d
nginx        NodePort    10.99.252.88   <none>        80:30608/TCP   44m
controlplane $ k get po nginx 
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          45m
controlplane $ k get po nginx  -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          45m   192.168.1.4   node01   <none>           <none>
```


**解釋**
當一個 Pod 嘗試訪問 Service 時，流量首先會被路由到 Service 的虛擬 IP 地址（在這個例子中是 10.99.252.88）。然後，**kube-proxy** 在該節點上的 NAT 表中查找與該 Service 相關的規則，並將流量重定向到 Service 後端的 Pod 的 IP 地址（在這個例子中是 192.168.1.6）。這種過程是透明的，從而實現了 Pod 和 Service 之間的連接。

**注意**

pod的IP是由cni決定，service的IP是由kube-apiserver決定，所以當你建立一個service時，kube-apiserver會為這個service分配一個IP，而這個IP是不會重複的，所以當你刪除一個service，這個IP就會被釋放，可以被其他service使用。

