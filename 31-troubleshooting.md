

/etc/kubernetes/manifests/裡的檔案都是透過volume mount進去的，要留意
> 例如ca.crt

journalctl -u kubelet : 用來看kubelet的log

node如果壞掉(not ready)，可以ssh到該node上，然後檢查container runtime 和 kubelet是否正常運作 (systemctl status docker.service, systemctl status kubelet.service)

kubelet的設定檔: /var/lib/kubelet/config.yaml

(client ca就是/etc/kubernetes/pki/ca.crt)

troubleshooting node:

 * 先在control plane上檢查node的狀態
 * ssh 進到壞掉的worker node上，檢查kubelet和container runtime的狀態
 * 若kubelet沒有跑起來，嘗試重啟

k8s troubleshooting 官網: 查"Troubleshooting"

網路相關問題:
 * CNI出錯
 * kube-proxy出錯
 * DNS出錯 

以上問題都先嘗試看看kubectl logs

> 留意: 有些pod會用到configmap，例如kube-proxy，所以可以確認configmap有沒有問題