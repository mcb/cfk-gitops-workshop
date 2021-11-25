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
kubectl get crds
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

With the CRDs in place, let's deploy Confluent Platform in a single node configuration. This can be done using the prepared YAML file in dev/confluent-platform-minikube.yaml

```
# Deploy the yaml
kubectl apply -f /home/ubuntu/code/cfk-workshop/dev/confluent-platform-minikube.yaml -n confluent
```

Check your current state with `kubectl get pods` or using a tool like `k9s`.

### Control Center Access

To fire up your Control Center you will need to check which service endpoint is for Control Center

```
kubectl get svc | grep controlcenter
NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                          AGE
controlcenter                ClusterIP      None             <none>        9021/TCP,7203/TCP,7777/TCP,7778/TCP                              80m
controlcenter-0-internal     ClusterIP      10.108.65.34     <none>        9021/TCP,7203/TCP,7777/TCP,7778/TCP                              80m
...
```

then start port forwarding to allow access

```
kubectl port-forward --address 0.0.0.0 svc/controlcenter-0-internal 9021:9021
```

Login to http://localhost:9021 to acces Control Center.

