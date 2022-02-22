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
jupyter server list
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
Alter `values.yaml`

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


## AWS Example

Create secret from a `key.json` file.
```sh
kubectl create secret generic aws-credentials --from-file="./config/key.json"
```

Or you can create a secret directly in the terminal:
```sh
kubectl create secret generic aws-credentials --from-literal=aws_access_key_id=<YOUR_KEY_ID> --from-literal=aws_secret_access_key=<YOUR_SECRET_KEY> 
```

Alter `values.yaml` to set your AWS credentials as environment variables
```yaml
# Allows you to load environment variables from kubernetes secret               
secret:                                                                         
  - envName: AWS_ACCESS_KEY_ID                                                  
    secretName: aws-credentials                                                 
    secretKey: aws_access_key_id                                                
  - envName: AWS_SECRET_ACCESS_KEY                                              
    secretName: aws-credentials                                                 
    secretKey: aws_secret_access_key   
```

And deploy the helm chart with `helm install` command shown above.

For the notebook to connect with AWS S3, you have to setup the correct spark configurations in your `.py` file. An example:
```python
from pyspark import SparkConf, SparkContext
from pyspark.sql import functions as f
from pyspark.sql import SparkSession

#spark configuration
conf = (
    SparkConf().set('spark.executor.extraJavaOptions','-Dcom.amazonaws.services.s3.enableV4=true')
    .set('spark.driver.extraJavaOptions','-Dcom.amazonaws.services.s3.enableV4=true')
    .set("spark.hadoop.fs.s3a.fast.upload", True)
    .set("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    .set('spark.jars.packages', 'software.amazon.awssdk:s3:2.17.133,org.apache.hadoop:hadoop-aws:3.2.0')
    .set('spark.hadoop.fs.s3a.aws.credentials.provider', 'com.amazonaws.auth.EnvironmentVariableCredentialsProvider')
)
sc=SparkContext(conf=conf).getOrCreate()

spark=SparkSession(sc)

df = spark.read.parquet("s3a:/<BUCKET-NAME>/<TABLE-NAME>/")

df.printSchema()
```

Make sure the credentials you passed as env variables do have access to the S3 bucket.

