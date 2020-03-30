# Udagram Microservice Deployment

### Background

Udagram is a simple cloud application developed alongside the Udacity Cloud Engineering Nanodegree. The app allows users to register and log into web client, post photos to the feed, and process photos using an image filtering microservice.

The project is composed of three pieces:

1. The Simple Frontend, an Ionic client web application which consumes the RestAPI Backend.
2. The RestAPI Feed Backend, a Node-Express feed microservice.
3. The RestAPI User Backend, a Node-Express user microservice.

The aim of the exercise is to build, test and deploy the microservice, using container technology, and CI/CD workflow.

### Installation

Ensure relevant dependencies are in place.

```bash
docker --version
aws --version
eksctl version
kubectl version --short --client
aws-iam-authenticator version
```

### Setup environment variables

Add following to the \$PATH.

```bash
export POSTGRESS_USERNAME=your postgress username;
export POSTGRESS_PASSWORD=your postgress password;
export POSTGRESS_DB=your postgress database;
export POSTGRESS_HOST=your postgress host;
export AWS_REGION=your aws region;
export AWS_PROFILE=your aws profile;
export AWS_BUCKET=your aws bucket name;
export JWT_SECRET=your jwt secret;
```

### Build docker images

Build local docker images for the project.

```bash
docker-compose -f ./udacity-c3-deployment/docker/docker-compose-build.yaml build --parallel
```

![build_docker_image](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/build_docker_image.png)

### Start docker containers

Spin up docker containers with following command.

```bash
docker-compose -f ./udacity-c3-deployment/docker/docker-compose.yaml up
```

![start_docker_container](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/start_docker_container.png)

### Test application

Make sure users can sign-up, login, post message, and attach pictures. Testing URL is at http://localhost:8100.

![test_web_app](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/test_web_app.png)

### Upload image to Dockerhub

For distribution over the internet, we publish container(s) to the Dockerhub.

```bash
docker-compose -f ./udacity-c3-deployment/docker/docker-compose-build.yaml push
```

![image_to_dockerhub](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/image_to_dockerhub.png)

### Setup Travis CI/CD

For CI/CD, we use Travis CI for this exercise. New commit to the github master branch will trigger new build process.

- Sign up Travis CI and connect Github repo to Trivis CI.
- Add .travis.yml to the root of the application.

```bash
language: minimal

services: docker

env:
  - DOCKER_COMPOSE_VERSION=1.23.2

before_install:
  - docker -v && docker-compose -v
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin
  - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
  - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
  - chmod +x ./kubectl
  - sudo mv ./kubectl /usr/local/bin/kubectl

install:
  - docker-compose -f udacity-c3-deployment/docker/docker-compose-build.yaml build --parallel
  - docker-compose -f udacity-c3-deployment/docker/docker-compose-build.yaml push
```

- Add microservice specific environment variables in TravisCI setting. Also add in credential for Dockhub to publish the built image.

![travis_env_vars](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/travis_env_vars.png)

- Commit and push changes to Github, and trigger build.

![travis_cicd_build](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/travis_cicd_build.png)

### Provision k8s cluster on AWS

To provision a k8s cluster on AWS, use eksctl command with script bellow.

```bash
eksctl create cluster \
--name "udagram"
--version 1.14 \
--region ap-northeast-1 \
--nodegroup-name udagram-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto
```

![aws_k8s_cluster](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/aws_k8s_cluster.png)

### Setup k8s configmaps and secrets

- Encode AWS-Credential in `aws_secret.yaml`. The pod will need AWS credential to access S3 bucket to save images that users upload.

```bash
cat ~/.aws/credentaisl | base64
```

- Save microservice specific configurations in `env-configmap.yaml`, e.g. AWS S3 bucket name, database name, database host, JWT secret, etc.

- Encode postgres database username and password in `env-secret.yaml`. The pod will need RDS credential to save user posts to the database.

```bash
echo -n 'POSTGRESS_USERNAME' | base64
echo -n 'POSTGRESS_PASSWORD' | base64
```

### Deploy configmap, secrets, and microservices

Deploy k8s configmaps and secrets.

```bash
kubectl apply -f aws-secret.yaml
kubectl apply -f env-secret.yaml
kubectl apply -f env-configmap.yaml
```

Deploy microservices. Specifically we deploy seperate services for frontend web app, NGINX reverse proxy, and two sets of backend APIs.

```bash
kubectl apply -f backend-feed-deployment.yaml
kubectl apply -f backend-feed-service.yaml
kubectl apply -f backend-user-deployment.yaml
kubectl apply -f backend-user-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f reverseproxy-deployment.yaml
kubectl apply -f reverseproxy-service.yaml
```

We then check status of pods , deplyments , and services.

```bash
kubectl get pods
kubectl get rs
kubectl get svc
kubectl get all
```

![aws_app_ready](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/aws_app_ready.png)

Finally we set up port forwarding to test drive the microservice.

```bash
kubectl port-forward service/reverseproxy 8080:8080
kubectl port-forward service/frontend 8100:8100
```

![k8s_port_forward](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/k8s_port_forward.png)

### Enable additional logging

The application is monitored by Amazon CloudWatch. Inside AWS EKS control plane, we can optionally activate various types of logging for purpose of debugging.

![aws_cloudwatch_logs](https://github.com/michlin0825/CDND-Udagram-Review/blob/master/screenshots/aws_cloudwatch_logs.png)

### Upgrade without disruption

For future releases of functionalities, we have choice of rolling update or blue/green deployment.

e.g. just deploy updated version

```bash
kubectl apply -f frontend-deployment
```

e.g. deploy two versions at the same time

```bash
kubectl apply -f frontend-deployment-alpha
kubectl apply -f frontend-deployment-beta
```

### Scale with elasticity

To cope with fluctuation of incoming traffic, we can scale up and down the number of servicing nodes in the cluster.

e.g. scale up

```bash
kubectl scale deployment/user --replicas=4
```

e.g. scale down

```bash
kubectl scale deployment/user --replicas=2
```
