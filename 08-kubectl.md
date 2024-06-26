# *Day8 Basic Concept* : `kubectl`基本應用整理

在前面的章節中，當我們要建立一個`k8s`物件時，我們不外乎就是透過兩種方式來達成:
   1. 先創建`yaml`，在使用`kubectl apply`建立物件
   2. 使用`kubectl`指令直接建立物件

在考試時，能用**最快的速度**滿足題目的需求是最重要的，所以除非指令無法直接達成題目要求，我們才會編輯`yaml`。

當然了，我們也不用花時間自己把yaml「刻」出來，一樣可以透過指令生成yaml樣本，再使用文字編輯器修改。

那麼今天的章節中，將針對前面幾天介紹的範圍，也就是`Pod`、`Deployment`、`Service`、`Namespace`，整理出一些常用的指令與技巧。

## 物件類別縮寫:
在`kubectl`指令中，我們常常需要指定物件類別，而縮寫可以大大加快敲指令的速度，以下是常見的縮寫:

物件類別|縮寫
---|---
pod|po
deployment|deploy
service|svc
namespace|ns


* 例如要查看所有`deployment`，以前我們會下達`kubectl get deployment`，而搭配縮寫後，只要敲:
```bash
kubectl get deploy
```

* 查看其他物件的縮寫:
```bash
kubectl api-resources
```

## 常用指令整理 --- get & describe

* 查看所有的物件，不論是`pod`、`deployment`、`service`等等
```bash
kubectl get all
```

* 查看某類物件
```bash
kubectl get <object-type>
```

* 計算有多少物件
```bash
kubectl get <object-type> | wc -l
# 結果要記得減1，因為第一行是header
```
or 
```bash
kubectl get <object-type> --no-headers | wc -l
```

* 查看特定`namespace`的物件
```bash
kubectl -n <namespace-name> get <object-type>
# 我習慣將`-n <namespace-name>`放在最前面，這樣比較不容易忘記
```
> 只要有任何操作有關於特定`namespace`，都可以加上`-n <namespace-name>`來指定

* 查看物件的`yaml`
```bash
kubectl get <object-type> <object-name> -o yaml
```
> "-o" 可以指定輸出的格式，例如: json、yaml、wide等等

* 查看物件的`label`
```bash
kubectl get <object-type> --show-labels
```

* 使用`selector`篩選特定的物件
```bash
kubectl get <object-type> -l <key=value>
# 多個label用逗號隔開，例如: -l <key1=value1,key2=value2>
```

* 搭配jsonpath查看特定欄位
```bash
kubectl get <object-type> -o jsonpath=<jsonpath-expression>
```

> 關於`jsonpath`的介紹與使用，可以參考[這裡](33-jsonpath.md)

* 查看物件的詳細資訊
```bash
kubectl describe <object-type> <object-name>
```

## 常用指令整理 --- `Pod`

* 建立一個`pod`
```bash
kubectl run <pod-name> --image <image-name>
```

* 建立一個`pod`並指定容器的`port`
```bash
kubectl run <pod-name> --image <image-name> --port <port-number>
```
* 建立一個 `pod`，並順便幫它建立一個`service`
```bash
kubectl run <pod-name> --image <image-name> --port <port-number> --expose
# service會與pod同名，預設為ClusterIP
```

* 建立一個`pod`，並指定`label`
```bash
kubectl run <pod-name> --image <image-name> -l <key=value>
# 如果有多個label，可以用逗號隔開，例如: -l <key1=value1,key2=value2>
```

* 更新`pod`的`image`
```bash
kubectl set image pod <pod-name> <container-name>=<new-image-name>
# container-name預設為pod-name
```

* 建立yaml樣本
```bash
kubeclt run <pod-name> --image <image-name>  --dry-run=client -o yaml > <filename.yaml>
# 其實只要是`kubectl create`或`kubectl run`都可以使用`--dry-run=client -o yaml`來產生yaml樣本 !
```

* 查看`pod`的log:
```bash
kubectl logs <pod-name>
```

* 查看`pod`的IP
```bash
kubectl get pod <pod-name> -o wide
```

## 常用指令整理 --- `Deployment`

* 建立一個`deployment`
```bash
kubectl create deploy <deploy-name> --image <image-name> --replicas <replicas-number>
# 不指定replicas的話，預設為1
```

* 建立一個`deployment`並指定容器的`port`
```bash
kubectl create deploy <deploy-name> --image <image-name> --replicas <replicas-number> --port <port-number>
```

* 改變`deployment`的`replicas`數量
```bash
kubectl scale deploy <deploy-name> --replicas <new-replicas-number>
```

* 更新`deployment`的`image`
```bash
kubectl set image deploy <deploy-name> <container-name>=<new-image-name>
# container-name預設為deploy-name
```

* 建立yaml樣本
```bash
kubectl create deploy <deploy-name> --image <image-name> --replicas <replicas-number> --dry-run=client -o yaml > <filename.yaml>
```

## 常用指令整理 --- `Service`

* 為pod/deploymnet建立一個`service`
```bash
kubectl expose <object-type> <object-name> --port <port-number> --target-port <target-port-number> --name <service-name> --type <service-type>
# 不指定target-port的話，預設與port相同
# 不指定type的話，預設為ClusterIP
# 不指定name的話，預設為object-name
```

> 舉例來說，要expose一個名為nginx-deploy的deployment，並且將service的type設為NodePort，那麼指令就會是:
```bash
kubectl expose deploy nginx-deploy --port 80 --name nginx-svc --type NodePort
```

** 建立yaml樣本
```bash
kubectl create svc <svc-type> <svc-name> --port <port-number> --target-port <port-number> --dry-run=client -o yaml > <filename.yaml>
```

## 基本技巧整理

1. 不要自己刻yaml，善用`--dry-run=client -o yaml`來產生yaml樣本
2. 如果遇到需要指定`namespace`的情況，用`-n <namespace-name>`來指定`namespace`
3. 善用-h來查看指令的使用方式，例如:
```bash
kubectl -h
kubectl run -h
kubectl create -h
kubectl expose -h
...
```
4. 考試是可以察官網的，只要在瀏覽器輸入`k8s.io`就會進到官網了











