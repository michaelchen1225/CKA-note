# 【附錄2】 Node Topology

> 我們在設定 Pod Affinity/Anti-Affinity 時，會設定一個 `topologyKey` 來劃分 Node Topology，以下用一個簡單的例子來說明：

假設我們有五個 Node，分別是 node01、node02、node03、node04、node05，它們各自有兩個 Label，如下表：

| Node   | Label  |
|--------|--------|
| node01 | zone=A, name=node01 |
| node02 | zone=A, name=node02 |
| node03 | zone=B, name=node03 |
| node04 | zone=C, name=node04 |
| node05 | region=X, number=05 |


當我們設定 topologyKey 為 `zone` 時，就是用 Node Label 中的 key `zone` 來劃分 Node Topology，這樣總共會分出四個 Node Topology：

| Node Topology | Node  |
|---------------|-------|
| zone=A        | node01、node02 |
| zone=B        | node03 |
| zone=C        | node04 |
| 沒有 zone key | node05 |


那如果我們設定 topologyKey 為 `name` 時，就是用 Node Label 中的 key `name` 來劃分 Node Topology，這樣總共會分出五個 Node Topology：

| Node Topology | Node  |
|---------------|-------|
| name=node01   | node01 |
| name=node02   | node02 |
| name=node03   | node03 |
| name=node04   | node04 |
| 沒有 name key | node05 |


我們將這個概念應用在 Inter-pod affinity/anti-affinity 上，就可以比較好理解了，舉例來說：

### Inter-pod affinity


沿用上面例子中的 Node Label，假設今天 inter-pod affinity 的設定如下：

* 篩選條件：有 `app=web` 的 Pod
* topologyKey：zone

假如只有 node01 上有 `app=web` 的 Pod，則 Pod 會被安排到 `zone=A` 這個 Node Topology 中，也就是 node01 或 node02 上。

那如果只有 node03 上有 `app=web` 的 Pod，則 Pod 會被安排到 `zone=B` 這個 Node Topology 中，也就是 node03 上。

### Inter-pod anti-affinity

再來看一個 inter-pod anti-affinity 的例子：

> 假設今天 inter-pod anti-Affinity 的設定如下：

* 篩選條件：有 `app=web` 的 Pod
* topologyKey：name

假如今天 node01、node02、node03 上都有 `app=web` 的 Pod，那 Pod 會被安排到 `name=node04` 或 `沒有 name key` 這兩個 Node Topology 中，也就是 node04 或 node05 上。

假如今天 node01、node02、node03、node04 上都有 `app=web` 的 Pod，那 Pod 會被安排到 `沒有 name key` 這個 Node Topology 中，也就是 node05 上。

----

**Reference**

* [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

* [解密 Assigning Pod To Nodes(下)](https://www.hwchiu.com/docs/2023/k8s-assigning-pod-2)