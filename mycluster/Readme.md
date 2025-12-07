#


## Install traefik controller
```bash
kubectl apply -f https://github.com/envoyproxy/gateway/releases/latest/download/install.yaml
kubectl -n envoy-gateway-system wait --for=condition=available --timeout=90s deployment/envoy-gateway-controller
```

## Install metallb
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
kubectl -n metallb-system wait --for=condition=available --timeout=90s deployment/controller

#Create a configmap for metallb with your desired IP address range
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.16.16.100-172.16.16.100

# Apply the configmap
kubectl apply -f metallb-configmap.yaml
```

## Deploy Argocd
```bash
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# update argocd service to NodePort
kubectl patch svc argocd-server -n argocd   -p '{"spec": {"type": "NodePort"}}'

# get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode
echo

hPZYkCQbxJwq19yq
f73J0QBy1yEJ52n0

## install cert-manager


# update argocd-server deployment to use tls secret
kubectl -n argocd patch deployment argocd-server \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'

# second update configmap to use tls
kubectl edit cm argocd -n argocd

apiVersion: v1
data:
  url: https://argocd.example.test
kind: ConfigMap
```

# Install server metrics
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl -n cert-manager get pods

### update metrics server for seflsign certificate
```yml
kubectl -n kube-system edit deployment metrics-server
spec:
  containers:
  - name: metrics-server
    args:
      - --cert-dir=/tmp
      - --secure-port=10250
      - --kubelet-insecure-tls
      - --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
      - --kubelet-use-node-status-port
      - --metric-resolution=15s

kubectl -n kube-system rollout restart deployment metrics-server
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system metrics-server-*
```


## helm installation for monitoring
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
kubectl create namespace monitoring


helm install central-monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f mycluster/central-monitoring-values.yaml

# for update

helm upgrade central-monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f mycluster/monitoring/central-monitoring-values.yaml
```

## Troubleshooting
- cordns issue
kubectl -n kube-system edit configmap coredns
- Replace /etc/resolv.conf with specific nameservers, e.g.:
forward . 8.8.8.8 8.8.4.4 {
    max_concurrent 1000
}
kubectl -n kube-system rollout restart deployment coredns

kubectl run -it --rm --restart=Never busybox --image=busybox sh
# inside pod
nslookup kubernetes.default
nslookup google.com

# Grafana troubleshooting
- kubectl -n monitoring exec -it central-monitoring-grafana-6fbd55f5bf-5lw76 -c grafana -- cat /etc/grafana/provisioning/datasources/datasource.yaml
- kubectl -n monitoring logs central-monitoring-grafana-67c78fbccf-t48x9 -c grafana --tail=40
- delete cm incase you have two instance of datasources
kubectl -n monitoring delete cm central-monitoring-kube-pr-grafana-datasource --force
