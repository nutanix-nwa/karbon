## Custum kubernetes cheat sheet

#### Apply one limit range on multiple namespaces
```console
for namespace in namespace1 namespace2 namespace3; do export NAMESPACE=$namespace ; cat limitrange.yaml | envsubst | kubectl apply -f -; done
```
