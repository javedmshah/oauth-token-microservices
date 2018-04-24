# About This Repository
This repository contains an environment variable-based configuration for the ForgeRock Identity Microservices on Kubernetes. The initial configuration can be used with the public bintray docker repository when deploying the early-access microservices docker images.

Use this repository as a starting point for your own custom configuration using environment variables for the ForgeRock Identity Microservices. For more information about specific use cases for Microservices, see the Microservices functional use cases guide in this repository.

Samples provided here have been tested with single kubernetes cluster in Minikube. 
Commands you need to run to deploy the containers are included here:

Deploy the Authentication microservice container :
kubectl create -f kube-authn-env-var.yaml

Deploy the Token Validation microservice container :
kubectl create -f kube-validation-env-var.yaml

Deploy the Token Exchange microservice container :
kubectl create -f kube-exchange-env-var.yaml

