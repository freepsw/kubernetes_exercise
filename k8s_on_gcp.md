
## Creating a Kubernetes Engine cluster

```
> gcloud auth list
> gcloud config list project
[core]
project = qwiklabs-gcp-ba88b7bdef68c2f6

> cloud config set compute/zone us-central1-a
Updated property [compute/zone].

> gcloud container clusters create my-k8s-cluster
kubeconfig entry generated for my-k8s-cluster.
NAME            LOCATION       MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
my-k8s-cluster  us-central1-a  1.11.8-gke.6    35.184.219.250  n1-standard-1  1.11.8-gke.6  3          RUNNING

```

## Get authentication credentials for the cluster
- After creating your cluster, you need to get authentication credentials to interact with the cluster.
```
> gcloud container clusters get-credentials my-k8s-cluster
Fetching cluster endpoint and auth data.
kubeconfig entry generated for my-k8s-cluster.
```

## Deploying an application to the cluster
- you'll run hello-app in your cluster.
- Kubernetes Engine uses Kubernetes objects to create and manage your cluster's resources.
- Kubernetes provides the Deployment object for deploying stateless applications like web servers.
- Service objects define rules and load balancing for accessing your application from the Internet.
```
# deploy Deployment object
> kubectl run hello-server --image=gcr.io/google-samples/hello-app:1.0 --port 8080
deployment.apps/hello-server created

# Now create a Kubernetes Service,
> kubectl expose deployment hello-server --type="LoadBalancer"
service/hello-server exposed

# Inspect the hello-server Service by running kubectl get:
> kubectl get service hello-server
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
hello-server   LoadBalancer   10.35.245.184   34.66.78.xxx   8080:30426/TCP   1m

```

## Clean up
```
> gcloud container clusters delete my-k8s-cluster
The following clusters will be deleted.
 - [my-k8s-cluster] in [us-central1-a]
```
