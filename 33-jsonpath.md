kubectl get <> -o jsonpath="{.items[*].status.phase}"

sort by 不用加.item[*]


custom column:
```bash
kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image'
```

```bash
