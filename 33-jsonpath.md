# 附錄: jsonpath

jsonpath是一種用來query json格式的語法，我們可以搭配`kubectl`來查詢特定欄位、排序、自訂輸出欄位等等。

在使用jsonpath以前，我們先來看看基本的json語法與其對應的yaml。

## json vs yaml

### key-value pair:

json

```json
{ "name": "Michael" }
```

yaml

```yaml
name: Michael
```


### Object:

json

```json
{
  "employee": {
    "name": "Michael",
    "age": "34"
  }
}
```

yaml

```yaml
employee:
  name: Michael
  age: 34
```

### Array:

json

```json
[ "Michael","Bob", "Alice"]
```

yaml
```yaml
- Michael
- Bob
- Alice
```

### Object值為Array:

json

```json
{
  "employee": [
    "Michael",
    "Bob",
    "Alice"
  ]
}
```

yaml

```yaml
employee:
  - Michael
  - Bob
  - Alice
```

### Array值為Object:

json

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

yaml

```yaml
- name: Michael
  age: 34
- name: Bob
  age: 30
```

### Array值為Array:

json

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

yaml

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

**jsonpath**:

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


