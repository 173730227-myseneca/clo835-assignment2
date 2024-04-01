# Ameya Kokatay's CLO835 Assignment 2

Please follow the steps listed below for this assignment.

- [Ameya Kokatay's CLO835 Assignment 2](#ameya-kokatays-clo835-assignment-2)
  - [Pre-requisites](#pre-requisites)
  - [1. Local K8s cluster](#1-local-k8s-cluster)
  - [2. Namespae and pod deployment](#2-namespae-and-pod-deployment)
  - [3. ReplicaSet Deployment](#3-replicaset-deployment)
  - [4. Applying Deployments](#4-applying-deployments)
  - [5. Creating services](#5-creating-services)
  - [6. Updating Deployment](#6-updating-deployment)
  - [7. ClusterIP vs NodePort](#7-clusterip-vs-nodeport)

## Pre-requisites

We start by copying config files and manifests into the EC2 instance we are using.

```bash
scp -i assignment2 kind.yaml init_kind.sh ec2-user@54.89.157.59:/tmp
scp -r -i assignment2 clo835-assignment2/* ec2-user@54.89.157.59:/tmp #Copying manifests into the ec2 instance
```

We then SSH into the instance and complete installing our pre-requisities.

```bash
ssh -i assignment2 ec2-user@54.89.157.59
cd /tmp
chmod +x init_kind.sh
./init_kind.sh

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 579130819361.dkr.ecr.us-east-1.amazonaws.com #authenticate to ECR to pull docker images
```

## 1. Local K8s cluster

We can see our local k8s kind cluster using `kubectl get nodes` and more details about it inclduing its IP address using `kubectl describe node kind-control-plane`

## 2. Namespae and pod deployment

We create two namespaces and deploy our pods into them with the following commands:

```bash
kubectl create namespace db
kubectl create namespace app

kubectl apply -f mysql_manifests/pod.yaml -n db
kubectl describe po mysql-pod -n db #we use this to update DB IP in the webapp manifests
kubectl apply -f webapp_manifests/pod.yaml -n app

kubectl get po -n db
kubectl get po -n app
```

a) Can both applications listen on the same port inside the container? Explain your answer.

> Yes, both applications in this scenario can listen on the same port since they are both running in separate pods and each pod will have the same port available without causing a conflict.

b) Connect to the server running in the application pod and get a valid response.

We setup port forwarding and use another terminal to make this connection and view the outcome using the following commands:

```bash
kubectl port-forward pod/webapp-pod 8080:8080 -n app
curl localhost:8080
```

c) Examine the logs of the invoked application to demonstrate the response from the server was reflected in the log file 

This can be achieved using the command: `kubectl logs webapp-pod -n app`

## 3. ReplicaSet Deployment

We use the following commands to deploy replicasets and view the outcome:

```bash
kubectl apply -f mysql_manifests/replicaset.yaml -n db
kubectl apply -f webapp_manifests/replicaset.yaml -n app
kubectl get rs -n db
kubectl get rs -n app
kubectl get pods -n db
kubectl get pods -n app
```

Here, we can see that the ReplicaSet maintains a desired state of 3 replica pods. Since we had one pod running already, it just spins up 2 new pods and the original pod remains intact. This indicates that the pod created in Step 2 is governed by the new ReplicaSet created.


## 4. Applying Deployments

We use the following commands to apply deployments and view the outcome:

```bash
kubectl apply -f mysql_manifests/deployment.yaml -n db
kubectl apply -f webapp_manifests/deployment.yaml -n app
kubectl get deployments -n db
kubectl get deployments -n app
kubectl get rs -n db
kubectl get rs -n app
```

From the output we get from the commands, we see that the pods in the original ReplicaSet have been replaced with pods created for the new deployment, showing us that the ReplicaSet created in Step 3 is not a part of the new deployment created.

## 5. Creating services

We use the following commands to apply deployments and view the outcome:

```bash
kubectl apply -f mysql_manifests/service.yaml -n db #after this is successfully deployed, change value in webapp deployment to mysql-service.db.svc.cluster.local
kubectl apply -f webapp_manifests/service.yaml -n app
kubectl get svc -n db
kubectl get svc -n app
curl 54.89.157.59:30000
```

## 6. Updating Deployment

We use the following commands to apply deployments and view the outcome:

```bash
kubectl rollout status deployment.apps/webapp-deployment -n app
kubectl rollout history deployment.apps/webapp-deployment -n app
```

## 7. ClusterIP vs NodePort

> We are using two different service types for our applications because both applications serve different purposes. Since our database should remain private, we use service type ClusterIP for it, ensuring that it can only be discovered internally and will remain secure. On the other hand, we want our webserver to be internet facing and externally accessible. We achieve this by exposing it using the NodePort service type.