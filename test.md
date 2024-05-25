## jsonpath in kubectl

我們直接來看幾個例子:

### 抓出所有pod的名稱:

* 先看看`-o json`的結果:
```bash
kubectl get pods -A -o json
```

* 了解了json的結構後，我們可以用jsonpath來抓取我們要的資料:

```bash
kubectl get pods -A -o jsonpath="{$.items[*].metadata.name}"
```

> 這裡jsonpath的根元素`$`可以省略不寫 :

```bash
kubectl get pods -A -o jsonpath="{.items[*].metadata.name}"
```
### 抓出所有pod的名稱與namespace:

```bash
kubectl get pods -A -o jsonpath="{.items[*].metadata.name}{.items[*].metadata.namespace}"
```
### 遞迴輸出所有pod的名稱:

* 但是剛剛的輸出結果並不容易閱讀，我們可以換行來輸出:
```bash
kubectl get pods -A -o jsonpath="{range .items[*]}{.metadata.name}{'\n'}{end}"
```

### 輸出所有pod的名稱，並自訂輸出欄位為`POD_NAME`:

* 為了更方便閱讀，我們可以自訂輸出欄位:

```bash
kubectl get pods -A -o custom-columns="POD-NAME:.metadata.name"
```

> **注意**: 使用自訂欄位時，.items需要省略


### 輸出所有pod的名稱與image，並且自訂欄位`POD_NAME`與`IMAGE`:

* 使用「,」來分隔不同的自訂欄位:

```bash
kubectl get pods -A -o custom-columns="POD_NAME:.metadata.name,IMAGE:.spec.containers[*].image"
```

### 抓出CPU資源為1或4的node:

```bash
kubectl get nodes -o jsonpath="{.items[?(@.status.allocatable.cpu=='1' || @.status.allocatable.cpu=='4')].metadata.name}"
```

### 列出`kube-scheduler-controlplane`所使用的image:

```bash
kubectl get pod kube-scheduler-controlplane -n kube-system -o jsonpath="{.spec.containers[*].image}"
```

或是:

```bash
kubectl get pod -A -o jsonpath="{.items[?(@.metadata.name=='kube-scheduler-controlplane')].spec.containers[*].image}"
```

### 根據pod對於CPU的需求，來排序pod:

* 先列出所有pod的name與對cpu的request(自訂欄位):
```bash
kubectl get pods -A -o custom-columns="POD_NAME:.metadata.name,REQUEST_CPU:.spec.containers[*].resources.requests.cpu"
```

* 使用`--sort-by`來排序:
```bash
kubectl get pods -A --sort-by="{.spec.containers[*].resources.requests.cpu}" -o custom-columns="POD_NAME:.metadata.name,REQUEST_CPU:.spec.containers[*].resources.requests.cpu"
```

> **注意**: `--sort-by`不需要加上.itmes[*]