## Custom kubernetes cheat sheet

#### Apply one limit range on multiple namespaces
```console
for namespace in namespace1 namespace2 namespace3; do export NAMESPACE=$namespace ; cat limitrange.yaml | envsubst | kubectl apply -f -; done
```

#### Apply label on some namespaces
```console
for namespace in $(kubectl get namespace|grep ns-test|awk {'print $1'}); do kubectl label ns $namespace environment=test; done
```

#### Create multiple sa in a global namespace
```console
for sa in sa1 sa2 sa3; do export SA_NAME=$sa ; export SA_NAMESPACE=sa-namespace ; cat sa.yaml | envsubst | kubectl apply -f -; done
```
