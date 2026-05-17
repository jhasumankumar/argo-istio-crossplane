# argo-istio-crossplane

## This document about istio and it's different features with example mostly using httpbin, argocd to deploy apps and crossplane to deploy using argocd

## Istio Features

| Feature | Manifest | Docs |
|---|---|---|
| Local Rate Limiting | [template/httpbin-local-rate-limit.yaml](template/httpbin-local-rate-limit.yaml) | [local-rate-limit.md](local-rate-limit.md) |

---

### Install argocd
- kubectl create namespace argocd
- kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/install.yaml
- kubectl get all -n argocd
- kubectl port-forward svc/argocd-server -n argocd 8080:443
- kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
- http://localhost:8080 user is admin and password can be retrieve from above command
```
# Login via REST — same endpoint the UI uses
PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo)

TOKEN=$(curl -sk https://localhost:8080/api/v1/session \
-H "Content-Type: application/json" \
-d "{\"username\":\"admin\",\"password\":\"$PASSWORD\"}" \
| python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

echo $TOKEN   # should print a JWT

export ARGOCD_SERVER=localhost:8080
export ARGOCD_AUTH_TOKEN=$TOKEN
export ARGOCD_OPTS="--insecure --grpc-web --http-retry-max 3"

# Test it
argocd app list
argocd cluster list

kubectl apply -f template/argocd-application-set.yaml 

kubectl get applicationset -n argocd
kubectl get applications   -n argocd -w

```


### Install crossplane
- helm repo add crossplane-stable https://charts.crossplane.io/stable
- helm repo update
- helm install crossplane \
  --namespace crossplane-system \
  --create-namespace crossplane-stable/crossplane