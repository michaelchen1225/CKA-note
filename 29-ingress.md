# ingress

建立service後，我們可以透過`ClusterIP`、`NodePort`、`LoadBalancer`來存取`Pod`。但是這些方式都是透過`IP`或`Port`來存取，對於使用者來說不太友好，而且當service的數量增加時，管理成本會越來越高，例如為了達到負載平衡而在每個service前面加上一個`LoadBalancer`，相當耗費金錢與資源。

如果說，我們能在nodec或cluster上建立一個**統一的入口**，並設定好**路由**規則，當使用者使用URL存取時就能將流量導向正確的service，這樣使用者只需要知道URL就能存取到服務。

這樣「各個service的統一入口」，就是`Ingress`。

除此之外，`Ingress`還有以下幾個功能可讓我們選擇:

 * **load balancing** : 將流量分散到不同的service上

  * **SSL termination** : 支援SSL，這樣使用者就能透過`https`存取服務，並且對於憑證的管理也會更加統一且方便。

我們用一張圖來看看「`Ingress`搭配load balancing功能，再轉發流量給service」的概念:
![ingress-example](29-1-ingress.png)

## Ingress controller

不過，`Ingress`只是一個抽象的規則，要讓`Ingress`生效，我們還需要一個`Ingress controller`。

`Ingress`制定的規則如果沒有`Ingress controller`來執行與控制，其實有跟沒有一樣(規則不會生效)。為了達成`Ingress`的目的，`Ingress controller`會透過kube-apiserver來監聽service與pod的變化，這樣才能根據`Ingress`的規則來做流量的轉發。

> 因為需要與kube-apiserver溝通，所以在安裝時通常會設定RBAC (等下安裝過程的輸出中會看到相關的訊息)

而`Ingress controller`有很多種，可以參考[官方文件](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers)的列表進行選擇。

我們可以依照不同需求在cluster中同時部署多種`Ingress controller`，並在設定`Ingress`時透過`Ingress class`來指定controller。

### 安裝Ingress controller

https://kubernetes.github.io/ingress-nginx/deploy/#quick-start
> 以下選用「[Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)」作為範例

* 安裝Ingress-Nginx Controller:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

* 確認一下是否有成功安裝:
```bash
kubectl get pods --namespace=ingress-nginx
```
```bash
# 應該要看到以下的輸出:
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-httbh        0/1     Completed   0          17m
ingress-nginx-admission-patch-ztmmr         0/1     Completed   1          17m
ingress-nginx-controller-7dcdbcff84-wl484   1/1     Running     0          17m
```

> 重點是`ingress-nginx-controller`這個pod有跑起來就好。

* 最後，將「default ingress class」設定為`nginx`:
```bash
kubectl edit ingressclass nginx
```
```yaml
# 修改如下:
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true" # 加入這個annotation
...
...(省略)...
```

> ingress class的概念有點像是storage class，用來區分不同的Ingress controller，這樣我們就能在Ingress中指定要使用哪一個controller。

**補充: 關於ingress-nginx-admission**

上面的輸出中那兩個completed的pod，他們的任務是建立ingress-admission，當任務完成當

admission是nginx-ingress-controller的一個webhook插件，每當有新的ingress被創建或更新時，會交由admission會檢查ingress的規則是否符合規則，整個流程如下

`ingress創建or更新`  --> `admission檢查`  -->  `符合規則`  -->  `controller執行`

* admission是以service的方式存在於cluster中，我們可以從下面指令的輸出中看到admission與contrller的關係:

```bash
kubectl describe svc -n ingress-nginx ingress-nginx-controller-admission | grep -i endpoint
# output: Endpoints:         192.168.1.6:8443
```
```bash
kubectl get po -n ingress-nginx ingress-nginx-controller-7dcdbcff84-wl484 -o wide | awk '{print $6}' 
# output:
# IP
# 192.168.1.6
```
> 其他關於admission與整個ingress-nginx-controller的運作原理，可參考[官方文件](https://kubernetes.github.io/ingress-nginx/how-it-works/)

## Ingress 實作

底下將透過實作來說明`Ingress`的設定方式，本次實作的目的如下:


建立兩個nginx pod，分別為`nginx-1`與`nginx-2`，並且透過`Ingress`來將流量分散到這兩個pod上。

  * 首先，我們先建立兩個nginx pod:
```bash
kubectl run nginx-1 --image=nginx --port=80
kubectl run nginx-2 --image=nginx --port=80
kubectl expose pod nginx-1 --port=80 --name=nginx-1-svc
kubectl expose pod nginx-2 --port=80 --name=nginx-2-svc
```

  * 然橫自訂一下index.html，這樣測試`Ingress`時比較好看出效果:
```bash
echo "Hello from nginx-1" > nginx-1.html
echo "Hello from nginx-2" > nginx-2.html
kubectl cp nginx-1.html nginx-1:/usr/share/nginx/html/index.html
kubectl cp nginx-2.html nginx-2:/usr/share/nginx/html/index.html
```

  * 查看一下pod的IP，等一下檢查`Ingress`的狀態時會用到:
```bash
kubectl get po nginx-1 nginx-2 -o wide
```
```bash
# 輸出如下
NAME      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
nginx-1   1/1     Running   0          9m46s   192.168.1.9    node01   <none>           <none>
nginx-2   1/1     Running   0          9m46s   192.168.1.10   node01   <none>           <none>
```

  * 接著，我們建立`Ingress`:
```yaml
# 底下關於「rewrite-target: /」這個annotation的意思，暫且先賣個關子，等一下會以實作來說明。
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /nginx-1
        pathType: Prefix
        backend:
          service:
            name: nginx-1-svc
            port:
              number: 80
      - path: /nginx-2
        pathType: Prefix
        backend:
          service:
            name: nginx-2-svc
            port: 
              number: 80
```

> 我們也可以透過kubectl來建立相同的`Ingress`:
```bash
kubectl create ingress ingress-nginx --rule='nginx-1=/nginx-1:80' --rule='nginx-2=/nginx-2:80'
```

  * 部署`Ingress`後查看一下情況:
```bash
kubectl apply -f ingress-nginx.yaml
kubectl describe ingress ingress-nginx
```
```bash
# 輸出如下
Name:             ingress-nginx
Labels:           <none>
Namespace:        default
Address:          
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /nginx-1   nginx-1-svc:80 (192.168.1.9:80) # 檢查IP是否符合預期?
              /nginx-2   nginx-2-svc:80 (192.168.1.10:80)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    13s (x2 over 4m19s)  nginx-ingress-controller  Scheduled for sync
```

  * 如果還記得上面的圖片，我們必須透過`Ingress controller`的service來存取`Ingress`，所以我們先來查看一下:
```bash
kubectl get svc -n ingress-nginx
```
```bash
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.103.189.113   <pending>     80:31098/TCP,443:32654/TCP   27m
ingress-nginx-controller-admission   ClusterIP      10.98.28.248     <none>        443/TCP                      27m
```
> 由於沒有設定SSL，所以我們走的是`80` port，其對應的nodePort為`31098`

  * 接著我們用curl來測試一下:
```bash
curl localhost:31098/nginx-1 && curl localhost:31098/nginx-2
```
```bash
# 成功的輸出如下:
Hello from nginx-1
Hello from nginx-2
```

  * 以上的存取紀錄也可以在各自service的log中看到:
```bash
kubectl logs svc/nginx-1-svc | grep "GET"
```
```bash
# 輸出如下
192.168.1.6 - - [03/Apr/2024:11:38:06 +0000] "GET / HTTP/1.1" 200 19 "-" "curl/7.68.0" "192.168.0.0"
```

以上就是相當簡單的`Ingress`測試，等下我們再來試試不同的`Ingress`功能。不過在此之前，先來填坑: 到底「nginx.ingress.kubernetes.io/rewrite-target: /」這個annotation是什麼意思?

  * 我們直接拿掉這個annotation，然後再次部署`Ingress`:
```bash
vim ingress-nginx.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  # annotations:
    # nginx.ingress.kubernetes.io/rewrite-target: /
...
...(省略)...
```

  * 部署後我們再用curl來測試一下:
```bash
curl localhost:31098/nginx-1 && curl localhost:31098/nginx-2
```

  * 結果系統丟回來「404 Not Found」:
```text
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.25.4</center>
</body>
</html>
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.25.4</center>
</body>
</html>
```

  * 我們找找service的log，看看是什麼原因:
```bash
kubectl logs svc/nginx-1-svc | grep "error"
```
```bash
# 輸出如下
2024/04/03 11:47:38 [error] 28#28: *2 open() "/usr/share/nginx/html/nginx-1" failed (2: No such file or directory), client: 192.168.1.6, server: localhost, request: "GET /nginx-1 HTTP/1.1", host: "localhost:31098"
```

**解釋**

當我們沒有加上annotation時，curl後面的URL會被轉為:

  (使用者輸入 --> 經ingress轉到的service --> service的pod)

* `localhost:31098/nginx-1` -> `nginx-1-svc:80/nginx-1` -> 192.168.1.9:80/nginx-1
* `localhost:31098/nginx-2` -> `nginx-2-svc:80/nginx-2` -> 192.168.1.10:80/nginx-2

這樣nginx就會去找`/usr/share/nginx/html/nginx-1`這個檔案，但是這個檔案並不存在，所以才會出現`404 Not Found`的錯誤。

* 因為ingress還沒改，我們用kubectl exec來驗證一下上面的說法(用nginx-1呼叫nginx-2):

```bash
# 嘗試呼叫nginx-2-svc:
k exec -it nginx-1 -- curl nginx-2-svc:80/index.html
k exec -it nginx-1 -- curl nginx-2-svc:80/nginx-2
```bash
# 輸出如下

