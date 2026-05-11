# argo-example
ArgoCD demo with http-echo Helm chart.

For my personal use - may be broken any time...

## Setup / bootstrap ArgoCD
```bash
export GITHUB_REPO='argo-example'
export GITHUB_REPO_BRANCH='main'  # or other branch
export CLUSTER_NAME='argocd-demo'

# When using Kind:
# Prereq to prevent 'too many open files' error:
# set these values via sysctl:
# ---
# fs.inotify.max_user_watches = 524288
# fs.inotify.max_user_instances = 8192
# fs.inotify.max_queued_events = 16384
# ---
# vi /etc/sysctl.d/99-kubernetes-inotify.conf  # add above to this file
# sysctl --system  # apply
# systemctl restart docker  # restart Docker for changes to take effect
export KIND_CLUSTER_NAME="$CLUSTER_NAME"
export KUBECONFIG="$HOME/.kube/kind-config-$KIND_CLUSTER_NAME"
kind create cluster --wait 5m  # Create a fresh Kubernetes cluster

# Install ArgoCD
kubectl create namespace argocd
kubectl apply --server-side=true -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# Takes some time until all Pods are running

# For access to the web ui, forward the port:
kubectl port-forward service/argocd-server -n argocd 8080:443
# Get admin password
ARGO_INITIAL_PASSWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```
## Helm deployments
### Create demo Helm app via web ui
Click Applications > Create application and fill in:
```
General:
  Application name = webapp-dev
  Project name = Default (click)
  Sync policy = Automatic (choose one)
    Check "Enable auto-sync"
    Check "Prune resources"
  Check "Auto-create namespace"
Source:
  Repository url = https://github.com/sven-dmlr/argo-examples.git
  Revision = master
  Path = (click) helm-webapp
Destination:
  Cluster url = https://kubernetes.default.svc (click)
  Namespace = webapp-dev
Helm:
  Values files = values-dev.yaml (click)
```
Then click "Create"

### Create demo Helm app via cli:
```bash
# Cli login
argocd login 127.0.0.1:8080
# prod app
argocd app create webapp-prod \
  --repo https://github.com/sven-dmlr/argo-examples.git \
  --path helm-webapp \
  --values values-prod.yaml \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace webapp-prod \
  --sync-policy manual \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true
```

### Create echo app with values from another repo
We use the definition from argo/echo-app.yaml:
```bash
kubectl apply -f argo/echo-app.yaml
```
Now watch the deployment in the ArgoCd web ui...

### Create echo app and pull Helm chart from external
Somewhat the same as above but simpler.
```bash
argocd app create http-echo2 \
  --project default \
  --repo https://haraldkoch.github.io/helm-charts \
  --helm-chart http-echo \
  --revision 0.3.4 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace http-echo2 \
  --sync-policy manual \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true
```

## Kustomize apps
Examples for deploying with Kustomize are quite stright forward:
```bash
# Dev app:
argocd app create webapp-kustom-dev \
  --repo https://github.com/sven-dmlr/argo-examples.git \
  --path kustom-webapp/overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev-from-cli \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true
```

```bash
# Prod app:
argocd app create webapp-kustom-prod \
  --repo https://github.com/sven-dmlr/argo-examples.git \
  --path kustom-webapp/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod-from-cli \
  --sync-policy manual \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true
```
