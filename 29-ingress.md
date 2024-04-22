# ingress

建立service後，我們就可以透過`ClusterIP`、`NodePort`、`LoadBalancer`等方式來存取`Pod`。以下為一個簡單的例子:

* 目前總共有四個deployments:
  1. `watch-photo`: 可以觀賞圖片
  2. `watch-video`: 可以觀賞影片
  3. `buy-clothes`: 可以購買衣服
  4. `buy-shoes`: 可以購買鞋子

* 為了讓使用者能夠存取，並希望每個deployment的pod達到附載平衡，因此我們在每個deployment上都配置了一個load balancer的service:
  1. `watch-photo-svc`: 10.0.0.1:80
  2. `watch-video-svc`: 10.0.0.2:80
  3. `buy-clothes-svc`: 10.0.0.3:80
  4. `buy-shoes-svc`: 10.0.0.4:80

如果使用者想要「看圖片」，情況如下圖:
![only-service](29-1-only-svc.png)

這樣的配置有以下缺點:

  * **造成使用者的不便**: 如上圖所示，使用者需知道service的IP與Port才能存取服務。如果未來service的數量增加，就等於要求使用者記住更多的IP與Port，強人所難。

  * **load balancer成本問題**: 每個service前面都要加上一個load balancer，而load balancer是需要花錢的，如果service的數量增加，成本也會增加。

要改善上面的缺點，比較理想的設計思路是:

> 由於服務目前主要針對兩個業務: `觀賞`(watch)與`購買`(buy)，因此設計讓使用者先決定要觀賞或購買，然後再選擇要觀賞什麼或購買什麼。

  * 如果使用者想要:
    * 「觀賞」: 連到`watch`的「入口」，然後再選擇要觀賞圖片或影片

    * 「購買」: 連到`buy`的「入口」，然後再選擇要購買衣服或鞋子

  * 因此我們在「入口」中這樣設計出路由規則:
    1. **/watch**(「觀賞」入口):
      * /photo -> watch-photo-svc
      * /video -> watch-video-svc
    
    2. **/buy**(「購買」入口):
      * /clothes -> buy-clothes-svc
      * /shoes -> buy-shoes-svc

而所謂的「入口」，就是`Ingress`。我們能在`Ingress`中設定路由規則，並將使用者的流量導向正確的service。

以上面的情境來說，如果使用者同樣想「看圖片」，情況如下圖:

![with-ingress](29-2-with-ingress.png)

所以，**設定路由規則**是`Ingress`的主要功能。但如果要讓`Ingress`生效，我們還需要`Ingress controller`。

## Ingress controller

`Ingress controller`有很多種，每種各有其特色與功能，可以參考[官方文件](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers)的列表進行選擇。

在一個cluster中可以同時存在多種`Ingress controller`，每種`Ingress controller`都有所屬的`Ingress class`與自己的service，兩者的功能如下:

* **Ingress class**

  前面提到，`Ingress`制定需要`Ingress controller`來執行才能生效，所以我們在設定`Ingress`時必須透過設定`Ingress class`來指定要使用哪一個`Ingress controller`。所以當哪天想要換其他的`Ingress controller`來執行相同的路由規則時，只需修改`Ingress`的`Ingress class`即可。

* **Ingress controller的service**

  當Ingress與Ingress controller建立起連繫後，使用者要透過domain name存取pod時，流程如下:
  
  1. 先經過`Ingress controller`的service抵達`Ingress controller`
  
  2. `Ingress controller`查找相對應的`Ingress`路由規則
  
  3. 流量從路由規則來到正確的service，最終到達提供服務的pod。

而為了達成`Ingress`所制定的規則，`Ingress controller`會透過kube-apiserver來監聽service與pod的變化，這樣才能根據`Ingress`的規則來做流量轉發。

> 因為需要與kube-apiserver溝通，所以在安裝Ingress controller時通常會設定RBAC (等下的安裝過程中可以自行留意一下)

除了讓`Ingress`生效外，`Ingress controller`還有其他的功能，常見的有:

  * load balancing : 附載平衡(解決了開頭提到的「load balancer成本問題」)
  * SSL termination : 支援SSL，這樣使用者就能透過https存取服務，並且對於憑證的管理也會更加統一且方便。

一下講了這麼多名詞，這裡一樣用開頭情境搭配圖示來說明:




接著，我們來實際安裝`Ingress controller`

### 安裝Ingress controller

> 以下選用「[Ingress-Nginx Controller](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)」作為範例

* 安裝Ingress-Nginx Controller:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

* 確認一下是否有成功安裝並執行:
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

> 上面的操作是將「nginx」設定為預設的Ingress class，若設定ingress時沒有指定ingress class，就會使用預設。

**補充: 關於ingress-nginx-admission**

上面的輸出中那兩個completed的pod，他們的任務是建立`ingress-admission`。

admission是nginx-ingress-controller的一個webhook插件，每當有新的ingress被創建或更新時，會交由admission會檢查ingress的規則是否符合規則，整個流程如下

`ingress創建or更新`  --> `admission檢查`  -->  `符合規則`  -->  `controller執行`

* `admission`是以service的方式存在於cluster中，我們可以從下面指令的輸出中看到admission與contrller的關係:

```bash
kubectl describe svc -n ingress-nginx ingress-nginx-controller-admission | grep -i endpoint
# output: Endpoints:         192.168.1.6:8443
# 表示該service連結到的pod的IP為192.168.1.6
```
```bash
kubectl get po -n ingress-nginx ingress-nginx-controller-7dcdbcff84-wl484 -o wide | awk '{print $6}' 
# output:
# IP
# 192.168.1.6
```
> 其他關於admission與整個ingress-nginx-controller的運作原理，可參考[官方文件](https://kubernetes.github.io/ingress-nginx/how-it-works/)

## Ingress 實作-1

底下將透過實作來說明`Ingress`的設定方式，本次實作的目的如下:

嘗試實做上面情境中的「buy」業務，不過我們先嘗試比較簡單的Ingress設定，熟練後再設定較複雜的。

  * 首先，我們先建立兩個pod與他們的service:
```bash
kubectl run watch-photo --image=nginx --port=80
kubectl run watch-video --image=nginx --port=80
kubectl expose pod watch-photo --port=80 --name=watch-photo-svc
kubectl expose pod watch-video --port=80 --name=watch-video-svc
```

  * 自訂一下index.html，這樣測試`Ingress`時比較好看出效果:
```bash
echo "Photo from watch-photo" > watch-photo.html
echo "Video from watch-video" > watch-video.html
kubectl cp watch-photo.html watch-photo:/usr/share/nginx/html/index.html
kubectl cp watch-video.html watch-video:/usr/share/nginx/html/index.html
```

  * 查看一下pod的IP，等一下檢查`Ingress`的狀態時會用到:
```bash
kubectl get po watch-photo watch-video -o wide
```
```bash
# 輸出如下
NAME          READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
watch-photo   1/1     Running   0          61s   192.168.1.7   node01   <none>           <none>
watch-video   1/1     Running   0          61s   192.168.1.8   node01   <none>           <none>
```

  * 接著，我們建立`Ingress`:
```yaml
# 底下關於「rewrite-target: /」這個annotation的意思，暫且先賣個關子，等一下會以實作來說明。
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-watch
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /watch-photo
        pathType: Prefix
        backend:
          service:
            name: watch-photo-svc
            port:
              number: 80
      - path: /watch-video
        pathType: Prefix
        backend:
          service:
            name: watch-video-svc
            port: 
              number: 80
```

> 我們也可以透過kubectl來建立相同的`Ingress`:
```bash
kubectl create ingress ingress-watch --rule='/watch-photo=watch-photo-svc:80' --rule='/watch-video=watch-video-svc:80' --annotation='nginx.ingress.kubernetes.io/rewrite-target=/'
```

  * 部署`Ingress`後查看一下情況:
```bash
kubectl apply -f ingress-watch.yaml
kubectl describe ingress ingress-nginx
```
```bash
# 輸出如下
Name:             ingress-watch
Labels:           <none>
Namespace:        default
Address:          
Ingress Class:    nginx # 上面沒指定ingress class，所以使用預設
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /watch-photo   watch-photo-svc:80 (192.168.1.7:80)
              /watch-video   watch-video-svc:80 (192.168.1.8:80)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    7s    nginx-ingress-controller  Scheduled for sync
```

  * 如果還記得上面的圖片，我們必須透過`Ingress controller`的service來存取`Ingress`，所以我們先來查看一下:

```bash
kubectl get svc -n ingress-nginx
```
```bash
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.109.140.197   <pending>     80:31001/TCP,443:31359/TCP   13m
ingress-nginx-controller-admission   ClusterIP      10.109.253.24    <none>        443/TCP                      13m
```
> 由於沒有設定SSL，所以我們走的是`80` port，其對應的nodePort為`31001`

  * 接著我們用curl來測試一下:
```bash
curl localhost:31001/watch-photo && curl localhost:31001/watch-video
```
```bash
# 成功的輸出如下:
Photo from watch-photo
Video from watch-video
```

  * 以上的存取紀錄也可以在各自service的log中看到:
```bash
kubectl logs svc/watch-video-svc | grep "GET"
```
```bash
# 輸出如下
192.168.1.6 - - [22/Apr/2024:12:48:16 +0000] "GET / HTTP/1.1" 200 23 "-" "curl/7.68.0" "192.168.0.0"
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



## rewrite: https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/learn/lecture/16827080#overview