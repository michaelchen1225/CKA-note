# Affinity
## Node Affinity
### RequiredDuringSchedulingIgnoredDuringExecution
### PreferredDuringSchedulingIgnoredDuringExecution
### 欄位設定
#### key
#### value
#### operator (In, NotIn, Exists, DoesNotExist)

# Taint & Toleration
## Taint(對象: Node)
### Taint effect
#### NoSchedule
#### PreferNoSchedule
#### NoExecute
## Toleration(對象: Pod)
### 欄位設定
#### key
#### value
#### operator (Exists, Equal)
#### effect (NoSchedule, PreferNoSchedule, NoExecute)
#### tolerationSeconds(只有在NoExecute時需要)



