# About This Repository
This repository contains an environment variable-based configuration for the ForgeRock Identity Microservices on Kubernetes. The initial configuration can be used with the public bintray docker repository when deploying the early-access microservices docker images.

Use this repository as a starting point for your own custom configuration using environment variables for the ForgeRock Identity Microservices. Sequence diagrams are included in this repository. Detailed use cases for microservices are described [here](https://forum.forgerock.com/2018/04/oauth2-forgerock-identity-microservices/) and [here](https://forum.forgerock.com/2018/04/token-exchange-and-delegation/).

Samples provided here have been tested with single kubernetes cluster in Minikube. 
Start minikube:
```shell
minikube start
```

Switch to using the minikube docker daemon:
```shell
eval $(minikube docker-env)
```
You can pull, and view images cached locally, view running containers, etc using the docker commands shown below.
However, it is strongly recommended to use the yaml files provided herein since these setup the microservices configuration using environment variables.

```shell
docker pull forgerock-docker-public.bintray.io/microservice/authn:1.0.0-SNAPSHOT
docker pull forgerock-docker-public.bintray.io/microservice/token-validation:1.0.0-SNAPSHOT
docker pull forgerock-docker-public.bintray.io/microservice/token-exchange:1.0.0-SNAPSHOT
```

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

The images should be pulled from the public bintray repository and show up with the command:
```shell
docker image list
```
```
REPOSITORY                                                         TAG                 IMAGE ID            CREATED             SIZE
forgerock-docker-public.bintray.io/microservice/token-exchange     1.0.0-SNAPSHOT      45773f9f05a1        2 days ago          112MB
forgerock-docker-public.bintray.io/microservice/authn              1.0.0-SNAPSHOT      16f9f07bdd0a        2 days ago          126MB
forgerock-docker-public.bintray.io/microservice/token-validation   1.0.0-SNAPSHOT      09cbfc694ddc        3 days ago          112MB
```
# AM instance
Artifacts in this repo expect a local AM instance running on http://openam.example.com:8080/openam
Amster export of this AM instance is included under https://github.com/javedmshah/forgerock-microservices/blob/master/amster-export.zip

# Token Exchange using AM as Policy

Please note that the configuration for token exchange relies on the AM policy artifacts defined in [this](https://github.com/javedmshah/token-exchange-microservice) repository. However the included amster configuration includes all those policy artifacts as well. So no separate download or setup is needed.
