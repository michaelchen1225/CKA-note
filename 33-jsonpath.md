# 附錄: jsonpath

jsonpath是一種用來query json格式的語法，我們可以搭配`kubectl`來查詢特定欄位、排序、自訂輸出欄位等等。

在使用jsonpath以前，我們先來看看基本的json語法與其對應的yaml。

## json vs yaml

### key-value pair:

json:

```json
{ "name": "Michael" }
```

yaml:

```yaml
name: Michael
```


### Object:

json:

```json
{
  "employee": {
    "name": "Michael",
    "age": "34"
  }
}
```

yaml:

```yaml
employee:
  name: Michael
  age: 34
```

### Array:

json:

```json
[ "Michael","Bob", "Alice"]
```

yaml:
```yaml
- Michael
- Bob
- Alice
```

### Object值為Array:

json:

```json
{
  "employee": [
    "Michael",
    "Bob",
    "Alice"
  ]
}
```

yaml:

```yaml
employee:
  - Michael
  - Bob
  - Alice
```

### Array值為Object:

json:

```json
[
  {
    "name": "Michael",
    "age": 34
  },
  {
    "name": "Bob",
    "age": 30
  }
]
```

yaml:

```yaml
- name: Michael
  age: 34
- name: Bob
  age: 30
```

### Array值為Array:

json:

```json
[
  {
    "age": [
      {
        "Michael": 34
      },
      {
        "Bob": 30
      }
    ]
  }
]
```

yaml:

```yaml
- age:
  - Michael: 34
  - Bob: 30
```

## jsonpath

以下為一份範例json檔:

```json
{ "store": {
    "book": [ 
      { "category": "reference",
        "author": "Nigel Rees",
        "title": "Sayings of the Century",
        "price": 8.95
      },
      { "category": "fiction",
        "author": "Evelyn Waugh",
        "title": "Sword of Honour",
        "price": 12.99
      },
      { "category": "fiction",
        "author": "Herman Melville",
        "title": "Moby Dick",
        "isbn": "0-553-21311-3",
        "price": 8.99
      },
      { "category": "fiction",
        "author": "J. R. R. Tolkien",
        "title": "The Lord of the Rings",
        "isbn": "0-395-19395-8",
        "price": 22.99
      }
    ],
    "bicycle": {
      "color": "red",
      "price": 19.95
    }
  }
}
```

> [範例來源](https://goessner.net/articles/JsonPath/)

底下示範幾個 jsonpath 的 query，你可以[點這裡](https://jsonpath.com/)跟著一起操作看看:

### 列出商店裡的所有商品:
```bash
$.store.*
```

### 列出所有書的價格:

```bash
$.store.book[*].price
```
或是
```bash
$..price
```

### 列出第二本書:

```bash
$.store.book[1]
```

### 列出第一本到第三本書:
```bash
$.store.book[0:3]
```

### 列出最後兩本書的:
```bash
$.store.book[-2:]
```

### 列出倒數第二本書:
```bash
$.store.book[-2:-1]
```

### 列出有isbn的書:
```bash
$.store.book[?(@.isbn)]
```

### 列出價格為12.99元的書:

```bash
$.store.book[?(@.price==12.99)]
```

**補充**

關於其他jsonpath的用法，請參考[這裡](https://github.com/json-path/JsonPath)

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


## REF
https://github.com/json-path/JsonPath

https://goessner.net/articles/JsonPath/

https://kubernetes.io/docs/reference/kubectl/jsonpath/
