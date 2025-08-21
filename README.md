# argocd-trino-install

Deploy the Trino Helm chart via Argo CD using an [app-of-apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps) setup.

![Alt text](argocd.png?raw=true)
![Alt text](argocd-trino.png?raw=true)

## Install Argo CD using a Helm umbrella chart
```
helm repo add argo-cd https://argoproj.github.io/argo-helm
cd argocd-trino-install
helm dep update charts/argo-cd
helm install argo-cd charts/argo-cd -n argo-cd --create-namespace
```

After sucessful deployment:
```
core@core-10920x:~/argocd-trino-install$ kubectl get all -n argo-cd
NAME                                                            READY   STATUS    RESTARTS   AGE
pod/argo-cd-argocd-application-controller-0                     1/1     Running   0          143m
pod/argo-cd-argocd-applicationset-controller-7b85784878-pkxrl   1/1     Running   0          143m
pod/argo-cd-argocd-redis-7d5fc4445c-l6p9s                       1/1     Running   0          143m
pod/argo-cd-argocd-repo-server-f7775f457-8fq44                  1/1     Running   0          143m
pod/argo-cd-argocd-server-5b459c4d6d-2flj4                      1/1     Running   0          143m

NAME                                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/argo-cd-argocd-applicationset-controller   ClusterIP   10.96.44.195     <none>        7000/TCP         143m
service/argo-cd-argocd-redis                       ClusterIP   10.109.112.236   <none>        6379/TCP         143m
service/argo-cd-argocd-repo-server                 ClusterIP   10.109.162.150   <none>        8081/TCP         143m
service/argo-cd-argocd-server                      ClusterIP   10.107.21.174    <none>        80/TCP,443/TCP   143m

NAME                                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argo-cd-argocd-applicationset-controller   1/1     1            1           143m
deployment.apps/argo-cd-argocd-redis                       1/1     1            1           143m
deployment.apps/argo-cd-argocd-repo-server                 1/1     1            1           143m
deployment.apps/argo-cd-argocd-server                      1/1     1            1           143m

NAME                                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/argo-cd-argocd-applicationset-controller-7b85784878   1         1         1       143m
replicaset.apps/argo-cd-argocd-redis-7d5fc4445c                       1         1         1       143m
replicaset.apps/argo-cd-argocd-repo-server-f7775f457                  1         1         1       143m
replicaset.apps/argo-cd-argocd-server-5b459c4d6d                      1         1         1       143m

NAME                                                     READY   AGE
statefulset.apps/argo-cd-argocd-application-controller   1/1     143m
```

## Access the Web UI
To access the Web UI, enable port-forwarding:
```
kubectl port-forward service/argo-cd-argocd-server -n argo-cd 8080:443
```
Then, go to `http://localhost:8080` on the web browser and accept the certificate.

Type in username `admin` and paste the random password generated during the installation:
```
kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Install the `argocd` CLI
```
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```
Log in to Argo CD from the command line:
```
core@core-10920x:~$ argocd login localhost:8080
WARNING: server is not configured with TLS. Proceed (y/n)? y
Username: admin
Password:
'admin:login' logged in successfully
Context 'localhost:8080' updated
```
Check version:
```
core@core-10920x:~$ argocd version
argocd: v3.1.0+03c4ad8
  BuildDate: 2025-08-13T21:03:54Z
  GitCommit: 03c4ad854c1e6d922a121cd40f505d9bc6e402da
  GitTreeState: clean
  GoVersion: go1.24.6
  Compiler: gc
  Platform: linux/amd64
argocd-server: v3.1.0+03c4ad8
  BuildDate: 2025-08-13T20:38:38Z
  GitCommit: 03c4ad854c1e6d922a121cd40f505d9bc6e402da
  GitTreeState: clean
  GoVersion: go1.24.6
  Compiler: gc
  Platform: linux/amd64
  Kustomize Version: v5.7.0 2025-06-28T07:00:07Z
  Helm Version: v3.18.4+gd80839c
  Kubectl Version: v0.33.1
  Jsonnet Version: v0.21.0
```

## Create the `trino` project
```
cat <<EOF | argocd proj create -f -
metadata:
  managedFields:
  - apiVersion: argoproj.io/v1alpha1
  name: trino
spec:
  destinations:
  - name: '*'
    namespace: trino
    server: '*'
  sourceRepos:
  - https://trinodb.github.io/charts/
EOF
```
List projects:
```
core@core-10920x:~$ argocd proj list
NAME     DESCRIPTION  DESTINATIONS  SOURCES                            CLUSTER-RESOURCE-WHITELIST  NAMESPACE-RESOURCE-BLACKLIST  SIGNATURE-KEYS  ORPHANED-RESOURCES  DESTINATION-SERVICE-ACCOUNTS
default               *,*           *                                  */*                         <none>                        <none>          disabled            <none>
trino                 *,trino       https://trinodb.github.io/charts/  <none>                      <none>                        <none>          disabled            <none>
```

## Create `root-app` and associated applications
Execute the following command:
```
core@core-10920x:~/argocd-trino-install$ helm template charts/root-app | kubectl apply -f -
Warning: metadata.finalizers: "resources-finalizer.argocd.argoproj.io": prefer a domain-qualified finalizer name including a path (/) to avoid accidental conflicts with other finalizer writers
application.argoproj.io/argo-cd created
application.argoproj.io/root-app created
application.argoproj.io/trino created
```
Lastly, list applications:
```
core@core-10920x:~/argocd-trino-install$ argocd app list
NAME              CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                     PATH              TARGET
argo-cd/argo-cd   https://kubernetes.default.svc  argo-cd    default  Synced  Healthy  Auto        <none>      https://github.com/mrdominguez/argocd-trino-install.git  charts/argo-cd    HEAD
argo-cd/root-app  https://kubernetes.default.svc  argo-cd    default  Synced  Healthy  Auto        <none>      https://github.com/mrdominguez/argocd-trino-install.git  charts/root-app/  HEAD
argo-cd/trino     https://kubernetes.default.svc  trino      trino    Synced  Healthy  Auto-Prune  <none>      https://trinodb.github.io/charts/                                          1.40.0
```
