# namespace -- 參考解答

* 建立`namespace`
```bash
$ kubectl create namespace practice
```

* 建立`deployment`
```bash
$ kubectl -n practice create deployment practice-deploy --image redis --replicas 2
```

* 查看是否符合要求
```bash
$ kubectl -n practice get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
practice-deploy   2/2     2            2           16s
```