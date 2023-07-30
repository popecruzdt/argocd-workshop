## ArgoCD + Dynatrace Workshop

### Terminal Prep
set git repo as env variable
```
export REPO=https://github.com/<username>/<this-repo>.git
```
clone repo to terminal
```
git clone $REPO
```
change directory to repo base
```
cd argocd-workshop
```

### ArgoCD Setup
set argocd namespace name as env variable, using only alpha and hyphens (no spaces, etc)
```
export ARGO_NS=<your-name>-argocd
```

#### July 2023 v2.7.9
create namespace
```
kubectl create namespace $ARGO_NS
```
deploy argocd default manifests
```
kubectl apply -n $ARGO_NS -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.9/manifests/install.yaml
```
download argocd cli
```
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v2.7.9/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

change UI server service from `ClusterIP` to `LoadBalancer` (external facing URL)
```
kubectl patch svc argocd-server -n $ARGO_NS -p '{"spec": {"type": "LoadBalancer"}}'
```

obtain external facing URL
```
kubectl get service -n $ARGO_NS argocd-server
```

obtain initial `password`
```
argocd admin initial-password -n $ARGO_NS
```

backup if the previous command fails
```
kubectl -n $ARGO_NS get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

connect to argocd using cli with user '`admin`', `password` from previous command, and `EXTERNAL-IP` URL
```
argocd login <EXTERNAL-IP>
```

change the admin password
```
argocd account update-password
```

login to the web ui in your browser
```
https://<EXTERNAL-IP>
```
#### Custom ArgoCD Namespace -> Modify ClusterRoleBinding Namespace
argocd expects to be installed into `argocd` namespace.  modify `ClusterRoleBinding` to match custom namespace.\
locate the (`2`) instances of the below config in `/argocd/2-7-9/argocd-cluster-role-binding.yaml` and replace value.
```
namespace: <your-name>-argocd
```
apply the `ClusterRoleBinding`
```
kubectl apply -f /argocd/2-7-9/argocd-cluster-role-binding.yaml
```

#### Configure Dynatrace Prometheus Scrape
update `argocd-metrics` kubernetes service with dynatrace annotations
```
kubectl apply -n $ARGO_NS -f argocd/2-7-9/argocd-metrics-service-with-dt.yaml
```
for reference:
```
annotations:
    metrics.dynatrace.com/scrape: 'true'
    metrics.dynatrace.com/port: '8082'
```

### Configure App

#### Using the UI approach
https://argo-cd.readthedocs.io/en/stable/getting_started/#creating-apps-via-ui

#### Using the CLI approach
modify kubectl context to use your `argocd` namespace
```
kubectl config set-context --current --namespace=$ARGO_NS
```
set env variable `APP_REPO`
```
export APP_REPO=https://github.com/<username>/<this-repo-name>.git
```
set env variable `APP_PATH`
```
export APP_PATH=app/simple-node-service/manifests
```
set env variable `APP_NS`
```
export APP_NS=<your-name>-app
```
create app namespace
```
kubectl create namespace $APP_NS
```
create app using argocd cli
```
argocd app create $APP_NS --repo $APP_REPO --path $APP_PATH --dest-server https://kubernetes.default.svc --dest-namespace $APP_NS
```