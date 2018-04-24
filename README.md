# About This Repository
This repository contains an environment variable-based configuration for the ForgeRock Identity Microservices on Kubernetes. The initial configuration can be used with the public bintray docker repository when deploying the early-access microservices docker images.

Use this repository as a starting point for your own custom configuration using environment variables for the ForgeRock Identity Microservices. For more information about specific use cases for Microservices, see the Microservices functional use cases guide in this repository.

Samples provided here have been tested with single kubernetes cluster in Minikube. 
Start minikube:
```shell
minikube start
```

Switch to using the minikube docker daemon:
```shell
eval $(minikube docker-env)
```
You can pull, and view images cached locally, view running containers, etc using the docker commands.

Commands you need to run to deploy the containers are included here.

Deploy the Authentication microservice container :
```shell
kubectl create -f kube-authn-env-var.yaml
```
Deploy the Token Validation microservice container :
```shell
kubectl create -f kube-validation-env-var.yaml
```
Deploy the Token Exchange microservice container :
```shell
kubectl create -f kube-exchange-env-var.yaml
```
