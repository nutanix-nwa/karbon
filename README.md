## Custom kubernetes cheat sheet

#### Apply one limit range on multiple namespaces
```bash
for namespace in namespace1 namespace2 namespace3; do \
export NAMESPACE=$namespace ; cat limitrange.yaml | envsubst | kubectl apply -f -; done
```

#### Apply label on some namespaces
```bash
for namespace in $(kubectl get namespace|grep ns-test|awk {'print $1'}); do \
kubectl label ns $namespace environment=test; done
```

#### Create multiple sa in a global namespace
```bash
for sa in sa1 sa2 sa3; do \
export SA_NAME=$sa ; export SA_NAMESPACE=sa-namespace ; cat sa.yaml | envsubst | kubectl apply -f -; done
```

#### Provide multiple sa admin access to a given namespace
```bash
for sa in sa1 sa2 sa3; do \
export SA_NAME=$sa ; export SA_NAMESPACE=sa-namespace ; export NAMESPACE=app-namespace ; \
cat sa-to-clusterrole-namespaced-admin.yaml | envsubst | kubectl apply -f -; done
```

#### Generate sa authentication config files
```bash
mkdir authentication-config; SA_NAMESPACE=sa-namespace ; \
for USERID in $(kubectl get serviceaccount -n $SA_NAMESPACE|grep -v "NAME\|default"|awk {'print $1'}); \
do cluster=$(kubectl config view --flatten --minify|grep "  cluster:"|awk -F: {'print $2'}|sed "s/ //g"); \
token=$(kubectl describe serviceaccount $USERID -n $SA_NAMESPACE|grep Tokens|awk {'print $2'}| \
awk {'print "kubectl get secret "$1" -n $SA_NAMESPACE -o \"jsonpath={.data.token}\" | base64 -d"'}|bash); \
kubectl config view --flatten --minify|sed "/    user:/c\    user: $USERID"| \
sed "/current-context:/c\current-context: $USERID@$cluster"|sed "/- name:/c\- name: $USERID"| \
sed "/token:/c\    token: $token"|sed "/namespace:/c\    namespace: $namespace"| \
sed -r "s/name: [a-z0-9]+@/name: $USERID@/" > authentication-config/${USERID}-authentication-config.yaml; done
```
