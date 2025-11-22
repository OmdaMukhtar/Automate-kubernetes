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
LyAA0D3XqHIECdpO

## install cert-manager

# update argocd-server deployment to use tls secret
kubectl -n argocd patch deployment argocd-server \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'

# second update configmap to use tls
apiVersion: v1
data:
  url: https://argocd.example.test
kind: ConfigMap
```



