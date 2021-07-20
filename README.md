## Custom kubernetes cheat sheet

#### Apply one limit range on multiple namespaces
```bash
for namespace in namespace1 namespace2 namespace3; do \
export NAMESPACE=$namespace ; cat limitrange.yaml | envsubst | kubectl apply -f -; done
```

#### Apply label on some namespaces
```bash
for namespace in $(kubectl get namespace|grep ns-test|awk {'print $1'}); do \
kubectl label ns $namespace environment=dev; done
```

#### Apply label on some nodes
```bash
for node in worker1 worker2; do \
kubectl label node $node role=infra; done

for node in worker3 worker4; do \
kubectl label node $node role=logging-monitoring; done
```

#### Apply taint on some nodes
```bash
for node in worker1 worker2; do \
kubectl taint node $node role=infra:NoSchedule; done

for node in worker3 worker4; do \
kubectl taint node $node role=logging-monitoring:NoSchedule; done
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

mkdir authentication-config; SA_NAMESPACE=globalaccount ; \
for USERID in $(kubectl get serviceaccount -n $SA_NAMESPACE|grep -v "NAME\|default"|awk {'print $1'}); do \
cluster=$(kubectl config view --flatten --minify|grep "  cluster:"| awk -F: {'print $2'}|sed "s/ //g"); \
token=$(kubectl describe serviceaccount $USERID -n $SA_NAMESPACE|grep Tokens|awk {'print $2'}|awk {'print "kubectl get secret "$1" -n $SA_NAMESPACE -o \"jsonpath={.data.token}\" | base64 -d"'}|bash); \
kubectl config view --flatten --minify| \
sed "/    user:/c\    user: $USERID"| \
sed "/current-context:/c\current-context: $USERID"| \
sed "/- name:/c\- name: $USERID"| \
sed "/^  name:/c\  name: $USERID"| \
sed "/^    cluster:/c\    cluster: $USERID"| \
sed "/token:/c\    token: $token"|sed "/namespace:/c\    namespace: $namespace"| \
sed -r "s/name: [a-z0-9]+@/name: $USERID@/" > authentication-config/${USERID}-authentication-config.yaml; \
done
```

#### Install Ingress Nginx
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace \
--set controller.hostNetwork=true --set controller.hostPort.enabled=true \
--set controller.kind=DaemonSet \
--set controller.tolerations[0].key=role,controller.tolerations[0].operator=Equal,controller.tolerations[0].value=infra,controller.tolerations[0].effect=NoSchedule \
--set controller.nodeSelector.role=infra

# Or
# helm show values ingress-nginx/ingress-nginx > ingress-nginx.values.yaml
# Edit the values
# vim ingress-nginx.values.yaml
# helm install ingress-nginx ingress-nginx/ingress-nginx -f ingress-nginx.values.yaml -n ingress-nginx --create-namespace
```

#### Install K8S Dashboard
```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard --create-namespace \
--set replicaCount=2 \
--set tolerations[0].key=role,tolerations[0].operator=Equal,tolerations[0].value=infra,tolerations[0].effect=NoSchedule \
--set nodeSelector.role=infra \
--set ingress.enabled=true \
--set ingress.hosts[0]=kubernetes-dashboard.home.local

# Or
# helm show values kubernetes-dashboard/kubernetes-dashboard > kubernetes-dashboard.values.yaml
# Edit the values
# vim kubernetes-dashboard.values.yaml
# helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f kubernetes-dashboard.values.yaml -n kubernetes-dashboard --create-namespace
```

#### Install ECK
```bash
helm repo add elastic https://helm.elastic.co
helm show values elastic/eck-operator > eck-operator.values.yaml
# Edit the values
vim eck-operator.values.yaml
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
kubectl create ns apps-central-logs
cat <<EOF | kubectl apply -n apps-central-logs -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: apps-central-logs
spec:
  version: 7.13.3
  nodeSets:
  - name: apps-central-logs
    count: 1
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-logging
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 200Gi
        storageClassName: default-storageclass
    podTemplate:
      spec:
        nodeSelector:
          role: logging-monitoring
        tolerations:
        - key: "role"
          operator: "Equal"
          value: "loggign-monitoring"
          effect: "NoSchedule"
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          resources:
            limits:
              memory: "4Gi"
              cpu: 2
            requests:
              memory: "4Gi"
              cpu: 2
          env:
          - name: ES_JAVA_OPTS
            value: "-Xms2g -Xmx2g"
EOF
cat <<EOF | kubectl apply -n apps-central-logs -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-apps-central-logs
spec:
  version: 7.13.3
  count: 1
  elasticsearchRef:
    name: apps-central-logs
  podTemplate:
    spec:
      nodeSelector:
        role: logging-monitoring
      tolerations:
      - key: "role"
        operator: "Equal"
        value: "logging-monitoring"
        effect: "NoSchedule"
      containers:
      - name: kibana
        resources:
          limits:
            memory: 1Gi
            cpu: 1
EOF
```
