# Pyspark Notebook Helm Chart

## Introduction
This repo provides 
the Kubernetes [Helm](https://helm.sh/) chart for deploying 
[Pyspark Notebook](https://hub.docker.com/r/jupyter/pyspark-notebook).

## Setup
1. Set up a kubernetes cluster
   - In a cloud platform of choice like [Amazon EKS](https://aws.amazon.com/eks), 
     [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine), 
     and [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/) OR
   - In local environment using [Minikube](https://minikube.sigs.k8s.io/docs/).
2. Install the following tools: 
   - [kubectl](https://kubernetes.io/docs/tasks/tools/) to manage kubernetes resources
   - [helm](https://helm.sh/docs/intro/install/) to deploy the resources based on helm charts. 
     Note, we only support Helm 3.
   
## Quickstart

Clone the repository

```(shell)
git clone https://github.com/A3Data/pyspark-helm.git
```

deploy Pyspark Notebook by running the following

```(shell)
helm dependency update ./pyspark-helm/Chart.yaml
helm install pyspark ./pyspark-helm/ --values ./pyspark-helm/values.yaml
```

Run `kubectl get all` to check whether all the pyspark resources are running. You should get a result similar to below.

```
NAME            READY   STATUS    RESTARTS   AGE
pod/pyspark-0   1/1     Running   0          9m18s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/pyspark            ClusterIP   10.110.1.129   <none>        8888/TCP,7777/TCP,2222/TCP   9m18s
service/pyspark-headless   ClusterIP   None           <none>        8888/TCP,7777/TCP,2222/TCP   9m18s

NAME                       READY   AGE
statefulset.apps/pyspark   1/1     9m18s
```

You can run the following to expose the notebook locally. 

```(shell)
kubectl port-forward svc/<release name> 8888:8888
```

You should be able to access the frontend via http://localhost:8888. 

## Get Token

```(shell)
kubectl exec -it pod/pyspark-0 -- bash
jupyter notebook list
```

## LoadBalancer

```sh
helm install pyspark ./pyspark-helm/ --values ./pyspark-helm/values.yaml --set service.type=LoadBalancer
```

## GCP Example

Create secret
```sh
kubectl create secret generic gcs-credentials --from-file="./config/key.json"
```
Alter values.yaml

```yaml
env: 
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: /mnt/secrets/key.json

extraVolumes: 
  - name: secrets
    secret:
      secretName: gcp-credentials

extraVolumeMounts:
  - name: secrets
    mountPath: "/mnt/secrets"
    readOnly: true 
```