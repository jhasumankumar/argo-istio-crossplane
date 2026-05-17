# argo-istio-crossplane

## Install argocd
- kubectl create namespace argocd
- kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/install.yaml
- kubectl get all -n argocd
- kubectl port-forward svc/argocd-server -n argocd 8080:443
- kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
- http://localhost:8080 user is admin and password can be retrieve from above command

## Install crossplane
- helm repo add crossplane-stable https://charts.crossplane.io/stable
- helm repo update
- helm install crossplane \
  --namespace crossplane-system \
  --create-namespace crossplane-stable/crossplane
- 