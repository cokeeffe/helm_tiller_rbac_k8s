# helm_tiller_rbac_k8s
Test deploying helm chart with a secure RBAC+Tiller setup on GKE

* This assumes gcloud and kubectl are installed and configured and if using GKE, a project is configured.
* We will use [redmine](https://github.com/kubernetes/charts/tree/master/stable/redmine) as an example application to setup as a HELM chart exists

## Install kubectx and kubens (optional)
This makes it easier to switch context and namespace

https://github.com/ahmetb/kubectx

## Setup GKE cluster (optional)
Note --no-enable-legacy-authorization to enable RBAC

```console
$ gcloud beta container clusters create redmine-rbac --zone us-east4-b --num-nodes 1 --project=colins-playground --no-enable-legacy-authorization

Creating cluster redmine-rbac...done.
Created [https://container.googleapis.com/v1/projects/colins-playground/zones/us-east4-a/clusters/redmine-rbac].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-east4-a/redmine-rbac?project=colins-playground
kubeconfig entry generated for redmine-rbac.
NAME          LOCATION    MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
redmine-rbac  us-east4-b  1.8.7-gke.1     35.230.162.142  n1-standard-1  1.8.7-gke.1   3          RUNNING

```

If using GKE, get credentials to the new clusters
```console
$ gcloud container clusters get-credentials redmine-rbac
Fetching cluster endpoint and auth data.
kubeconfig entry generated for redmine-rbac.
```

and add permissions to your account
```console
$ kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

## Switch Context
We can use local K8s (docker-for-desktop), minikube or GKE
```console
$ kubectx
docker-for-desktop
gke_colins-playground_us-east4-a_hello-world-rbac
minikube

$ kubectx gke_colins-playground_us-east4-b_redmine-rbac
Switched to context "gke_colins-playground_us-east4-b_redmine-rbac".
```
## Install Helm-Secure plugin

```console
 helm plugin install https://github.com/michelleN/helm-secure-tiller
 ```

## Create Namespace and install Tiller  
```console
$ kubectl create namespace redmine-app-ns
# install tiller in a namespace like this:
$ helm init --tiller-namespace redmine-app-ns
```

## Switch to the namespace
```console
$ kubens redmine-app-ns
Context "gke_colins-playground_us-east4-a_redmine-rbac" modified.
Active namespace is "redmine-app-ns".
```

Use secure-tiller plugin to apply an RBAC policy to the namespace. Create a policy per namespace as it's safer to install tiller per namespace. Using example-profiles/dev-team/ in this example:

```console
$ helm secure-tiller --namespace redmine-app-ns example-profiles/redmine-app/
serviceaccount "redmine-app" created
role "redmine-app" created
rolebinding "tiller-binding-redmine-app" created
```

## Make a deployment using a HELM chart

```console
$ helm install --tiller-namespace redmine-app-ns --name my-release stable/redmine
```

## Summary of Commands
```console
gcloud beta container clusters create redmine-rbac --zone us-east4-b --num-nodes 1 --project=colins-playground --no-enable-legacy-authorization
gcloud container clusters get-credentials redmine-rbac
kubectl create clusterrolebinding cluster-admin-binding   --clusterrole cluster-admin   --user $(gcloud config get-value account)
kubectl create namespace redmine-app-ns
kubens redmine-app-ns
helm init --tiller-namespace redmine-app-ns
helm secure-tiller --namespace redmine-app-ns example-profiles/redmine-app/
helm install --tiller-namespace redmine-app-ns --name my-release stable/redmine
```
