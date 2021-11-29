# Confluent for Kubernetes GitOps Workshop Sample Repo

## Install Confluent for Kubernetes

### Minikube for development environment

Start the minikube cluster

```
minikube start --kubernetes-version=v1.20.9
```

## Install Confluent for Kubernetes Operator

After checking that minikube is up and running, we'll install the Confluent for Kubernetes (CFK) Operator in a specific namespace "confluent".

```
# Create namespace to use
kubectl create ns confluent

# Add the Confluent Helm repository. Helm is used to package the Confluent for Kubernetes(CFK) Operator and CRDs.
helm repo add confluentinc https://packages.confluent.io/helm

# Install CFK Operator into the confluent namespace
helm install cfk-operator confluentinc/confluent-for-kubernetes -n confluent

# Check the install  is successful, list the installed chart
helm list -n confluent
NAME        	NAMESPACE	REVISION	UPDATED                            	STATUS  	CHART                           	APP VERSION
cfk-operator	confluent	1       	2021-11-25 14:48:28.67959 +0100 CET	deployed	confluent-for-kubernetes-0.304.2	2.2.0

# The Helm chart deploys the Confluent for Kubernetes  (CFK) Operator as a pod. You should see it up and running.
kubectl get pods -n confluent
NAME                                  READY   STATUS    RESTARTS   AGE
confluent-operator-849956574b-n28jj   1/1     Running   0          91m
```


CustomResourceDefinition (CRD) resource allows you to define custom resources. Confluent for Kubernetes includes CRDs for each component as well as for RBAC and topics.

```
kubectl get crds -n confluent
NAME                                          CREATED AT
clusterlinks.platform.confluent.io            2021-11-25T13:48:26Z
confluentrolebindings.platform.confluent.io   2021-11-25T13:48:26Z
connectors.platform.confluent.io              2021-11-25T13:48:26Z
connects.platform.confluent.io                2021-11-25T13:48:26Z
controlcenters.platform.confluent.io          2021-11-25T13:48:26Z
kafkarestclasses.platform.confluent.io        2021-11-25T13:48:26Z
kafkarestproxies.platform.confluent.io        2021-11-25T13:48:26Z
kafkas.platform.confluent.io                  2021-11-25T13:48:26Z
kafkatopics.platform.confluent.io             2021-11-25T13:48:26Z
ksqldbs.platform.confluent.io                 2021-11-25T13:48:26Z
migrationjobs.platform.confluent.io           2021-11-25T13:48:26Z
schemaregistries.platform.confluent.io        2021-11-25T13:48:26Z
schemas.platform.confluent.io                 2021-11-25T13:48:26Z
zookeepers.platform.confluent.io              2021-11-25T13:48:26Z
```

With the CRDs in place, let's deploy Confluent Platform in a single node configuration. This can be done using the prepared YAML file in [clusters/minikube/confluent/confluent-platform-minikube.yaml](clusters/minikube/confluent/confluent-platform-minikube.yaml)

```
# Deploy the yaml
kubectl apply -f dev/confluent-platform-minikube.yaml -n confluent
```

Check your current state with `kubectl get pods` or using a tool like `k9s`.

### Control Center Access

To fire up your Control Center you will need to check which service endpoint is for Control Center

```
kubectl get svc -n confluent | grep controlcenter
NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                          AGE
controlcenter                ClusterIP      None             <none>        9021/TCP,7203/TCP,7777/TCP,7778/TCP                              80m
controlcenter-0-internal     ClusterIP      10.108.65.34     <none>        9021/TCP,7203/TCP,7777/TCP,7778/TCP                              80m
...
```

then start port forwarding to allow access

```
kubectl port-forward -n confluent --address 0.0.0.0 svc/controlcenter-0-internal 9021:9021
```

Login to http://localhost:9021 to acces Control Center.


## Install fluxCD

Install fluxCD via Homebrew (on macOS) or similiar:

```
brew install fluxcd/tap/flux
```

and check if the preflight checks pass

```
flux check --pre
► checking prerequisites
✔ Kubernetes 1.20.2 >=1.19.0-0
✔ prerequisites checks passed
```

### Export your Github personal token

Go to github.com, navigate to Settings -> Developer -> Personal access token and generate a token with repo rights

```
export GITHUB_TOKEN=<github_token>
export GITHUB_USER=<github_username>
```

Bootstrap flux
```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=cfk-gitops \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
  ```
  
  and the output will be similiar to 
  ```
  ► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/mcb/cfk-gitops.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed sync manifests to "main" ("7a6e4d673fe7a5e50a9c0085c2cc875f23ec6d4a")
► pushing component manifests to "https://github.com/mcb/cfk-gitops.git"
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ configured deploy key "flux-system-main-flux-system-./clusters/minikube" for "https://github.com/mcb/cfk-gitops"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("b95d6f3de4e7570184e9fab226a8afee4dabf92c")
► pushing sync manifests to "https://github.com/mcb/cfk-gitops.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```
  
Flux will generate a separate namespace and create 4 resources in it, let's have a look what has been generated

```
kubectl get pods -n flux-system
NAME                                      READY   STATUS    RESTARTS   AGE
helm-controller-66b59bb6dc-z97nb          1/1     Running   0          24m
kustomize-controller-7987d6697f-rnp8p     1/1     Running   0          24m
notification-controller-945795558-hp22l   1/1     Running   0          24m
source-controller-b5bd68987-dpt5q         1/1     Running   0          24m
```

Also, you will notice a bunch of files in the clusters/minikube directory. This is where the configuration for flux-system goes, we will make use of the separation by namespaces. 

To add a new namespace, simply add a new file to a new directory

```
# create directory to hold namespace configuration
mkdir clusters/minikube/confluent

# create namespace.yaml

cat <<EOF > clusters/minikube/confluent/namespace.yaml                                                                                                      main
---
apiVersion: v1
kind: Namespace
metadata:
  name: confluent
EOF
```

Commit and push to github and see flux configuring the namespace for you. If you cannot see the changes as you might have already the confluent namespace from the previous step, you may inspect the kustomize-controller logs

```
# check the kusztomize-controller logs, your id will be slightly different (see step above)
kubectl logs -n flux-system kustomize-controller-7987d6697f-rnp8p | grep confluent
{"level":"info","ts":"2021-11-29T13:29:05.121Z","logger":"controller.kustomization","msg":"server-side apply completed","reconciler group":"kustomize.toolkit.fluxcd.io","reconciler kind":"Kustomization","name":"flux-system","namespace":"flux-system","output":{"CustomResourceDefinition/alerts.notification.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/buckets.source.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/gitrepositories.source.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/helmcharts.source.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/helmreleases.helm.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/helmrepositories.source.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/kustomizations.kustomize.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/providers.notification.toolkit.fluxcd.io":"unchanged","CustomResourceDefinition/receivers.notification.toolkit.fluxcd.io":"unchanged","Namespace/confluent":"configured","Namespace/flux-system":"unchanged"}}
```

This will reveal that the `"Namespace/confluent":"configured"` has been successfully triggered.

Now that we have a configuration for the namespace, let's add our confluent-platform-minikube.yaml to be managed by flux. This can be done by simply adding it to the confluent directory, we created earlier. You will now see more configurations happening in the kustomize-controller logs

```
~ % kubectl logs -n flux-system kustomize-controller-7987d6697f-rnp8p | grep kafka
{"level":"info","ts":"2021-11-29T13:33:09.563Z","logger":"controller.kustomization","msg":"server-side apply completed","reconciler group":"kustomize.toolkit.fluxcd.io","reconciler kind":"Kustomization","name":"flux-system","namespace":"flux-system","output":{"ClusterRole/crd-controller-flux-system":"unchanged","ClusterRoleBinding/cluster-reconciler-flux-system":"unchanged","ClusterRoleBinding/crd-controller-flux-system":"unchanged","Connect/confluent/connect":"configured","ControlCenter/confluent/controlcenter":"configured","Deployment/flux-system/helm-controller":"unchanged","Deployment/flux-system/kustomize-controller":"unchanged","Deployment/flux-system/notification-controller":"unchanged","Deployment/flux-system/source-controller":"unchanged","GitRepository/flux-system/flux-system":"unchanged","Kafka/confluent/kafka":"configured","KsqlDB/confluent/ksqldb":"configured","Kustomization/flux-system/flux-system":"unchanged","NetworkPolicy/flux-system/allow-egress":"unchanged","NetworkPolicy/flux-system/allow-scraping":"unchanged","NetworkPolicy/flux-system/allow-webhooks":"unchanged","SchemaRegistry/confluent/schemaregistry":"configured","Service/flux-system/notification-controller":"unchanged","Service/flux-system/source-controller":"unchanged","Service/flux-system/webhook-receiver":"unchanged","ServiceAccount/flux-system/helm-controller":"unchanged","ServiceAccount/flux-system/kustomize-controller":"unchanged","ServiceAccount/flux-system/notification-controller":"unchanged","ServiceAccount/flux-system/source-controller":"unchanged","Zookeeper/confluent/zookeeper":"configured"}}
```
