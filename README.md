# vproapp-devops-cicd

> Production-minded CI/CD reference for a Java web application — containerized, pipeline-ready, and deployable to your own AWS EKS cluster.

## Overview

`vproapp-devops-cicd` is a practical repository that demonstrates how to build, containerize, test, and deploy a Java web application using Docker, CI/CD (Jenkins), and Kubernetes on AWS (EKS). It's intended as a blueprint you can adapt and extend for real projects.

## Features

- End-to-end examples: local Docker + Docker Compose, CI pipeline (`Jenkinsfile`), and Kubernetes manifests for cloud deployment.
- Services included: application, database, RabbitMQ, and optional Elasticsearch utilities.
- Opinionated, yet adaptable: default configurations are provided so you can get started quickly and then customize for your environment.

## Architecture (high level)

- Developer: build and test with Maven locally.
- Build system: Jenkins pipeline runs unit tests, builds Docker images, and pushes images to a registry (e.g., AWS ECR).
- Runtime: Kubernetes cluster (EKS) runs application, DB and messaging services with provided manifests.

## Prerequisites

- Java JDK and Maven
- Docker & Docker Compose
- Git
- AWS account with permissions to create ECR repositories, EKS clusters, and IAM roles
- AWS CLI configured with appropriate credentials
- `eksctl` (or `eksctl` alternatives) to create/manage EKS clusters
- `kubectl` configured to access your cluster
- Jenkins (or other CI system) for pipeline automation

## Quick local workflow

1. Build the application artifact:

```powershell
mvn clean package -DskipTests
```

2. Build Docker images locally (optional):

```powershell
docker build -t vproapp:local -f Docker-files/app/Dockerfile .
docker build -t vproweb:local -f Docker-files/web/Dockerfile .
```

3. Start with Docker Compose for local testing:

```powershell
docker-compose up --build
```

4. Stop and clean up when done:

```powershell
docker-compose down --volumes
```

## Publish images to AWS ECR (example)

1. Create an ECR repository (one-time):

```powershell
aws ecr create-repository --repository-name vproapp --region us-east-1
```

2. Authenticate Docker to ECR and push:

```powershell
$ecrUri = "<your-account-id>.dkr.ecr.us-east-1.amazonaws.com/vproapp"
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-east-1.amazonaws.com
docker tag vproapp:local $ecrUri:latest
docker push $ecrUri:latest
```

Replace `<your-account-id>` and `us-east-1` with your AWS account ID and region.

## Deploy to a self-managed Kubernetes cluster on EC2 (3 nodes)

If you're running a Kubernetes cluster on three EC2 instances (self-managed control plane + workers) rather than EKS, follow these high-level steps. The example below uses `kubeadm` (vanilla Kubernetes). Alternatives include `kops`, `k3s`, or provisioning via Terraform.

1) Provision three EC2 instances

- Create one control-plane (master) instance and two worker instances (or 3 nodes in any role depending on HA needs).
- Recommended AMI: Ubuntu 22.04 / Amazon Linux 2. Configure security groups to allow SSH (port 22), Kubernetes API (6443) on control plane, and node ports as required.
- Ensure each node has an IAM role or credentials to pull from ECR, or plan to use imagePullSecrets (see notes below).

2) Prepare nodes (run on all nodes as root or via sudo)

```powershell
# Update & install container runtime (example using Docker)
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update && sudo apt-get install -y docker-ce

# Disable swap (required by kubeadm)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

3) Install Kubernetes tools (on all nodes)

```powershell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

4) Initialize the control plane (run on control-plane node)

```powershell
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
# follow kubeadm output: copy the `kubeadm join ...` command for worker nodes

# Configure kubectl for the ubuntu user (or your user)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5) Install a CNI (network plugin) — e.g., Calico

```powershell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

6) Join worker nodes (run the `kubeadm join ...` command from step 4 on each worker)

7) Verify cluster

```powershell
kubectl get nodes
kubectl get pods -A
```

8) Ensure your nodes can pull images from ECR

Option A — Node IAM role: attach an IAM role to each EC2 node with permissions to read ECR and the node will be able to pull images directly (recommended for production).

Option B — Docker login on nodes (for small/test clusters): run docker login on each node using an ECR-auth token before pods attempt to pull images.

```powershell
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-east-1.amazonaws.com
```

Option C — `imagePullSecrets`: create a Kubernetes secret with ECR credentials and reference it in your deployment manifests.

9) Deploy application manifests

- Update `vproappdep.yml`, `rmq-dep.yml`, and `vprodbdep.yml` to use your ECR image URIs, for example:

```yaml
image: <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/vproapp:latest
```

- Apply the manifests from your workstation (with kubeconfig) or from the control-plane node:

```powershell
kubectl apply -f vproappdep.yml
kubectl apply -f rmq-dep.yml
kubectl apply -f vprodbdep.yml
```

10) Monitor rollout

```powershell
kubectl rollout status deployment/vproapp
kubectl get pods -w
kubectl logs deployment/vproapp -c <container-name>
```

Notes & alternatives

- Using `kops` or `kubespray` can simplify creating a production-grade cluster across EC2 instances (networking, autoscaling, upgrades). Consider these for larger environments.
- For small footprints, `k3s` provides a lightweight Kubernetes distribution that is quick to provision on EC2.
- Ensure the control plane is backed up and you plan for etcd backups if you run your own control plane nodes.

## CI/CD & Jenkins (self-managed cluster)

If Jenkins will deploy to your self-managed cluster, choose one of these deployment strategies:

- Jenkins agent with `kubectl` and valid `kubeconfig` that can talk to the cluster (secure the kubeconfig). Pipeline runs `kubectl apply`.
- Jenkins runs `ssh` to the control-plane node and executes `kubectl apply` there (simpler but requires SSH automation).

Example pipeline considerations:

- Ensure the Jenkins agent can authenticate to ECR and your cluster. For nodes pulling images, use node IAM roles or `imagePullSecrets`.
- Replace `eksctl` steps in the pipeline with the `kubectl apply` steps above or remote SSH commands.

Example deploy step (run by a Jenkins agent with `kubectl` configured):

```groovy
stage('Deploy') {
  steps {
    sh 'kubectl apply -f vproappdep.yml'
    sh 'kubectl rollout status deployment/vproapp'
  }
}
```


## CI/CD with Jenkins (recommended pipeline)

High-level pipeline stages (see `Jenkinsfile`):

- Checkout
- Build and unit test (`mvn clean package`)
- Build Docker images
- Authenticate and push to container registry (AWS ECR)
- Deploy to cluster (`kubectl apply` or Helm)

Essential Jenkins configuration:

- Add AWS credentials (access key/secret) and Docker registry credentials as Jenkins credentials.
- Install necessary Jenkins plugins: Pipeline, Kubernetes CLI, Amazon ECR, Docker Pipeline (as needed).
- Ensure the Jenkins agent has `docker`, `kubectl` and `aws` CLI available (or run these steps in a dockerized agent image).

Example pipeline snippet (conceptual):

```groovy
pipeline {
  agent any
  environment {
    ECR_REGISTRY = '<your-account-id>.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_NAME = 'vproapp'
  }
  stages {
    stage('Build') { steps { sh 'mvn clean package -DskipTests' } }
    stage('Build Image') { steps { sh "docker build -t $ECR_REGISTRY/$IMAGE_NAME:latest -f Docker-files/app/Dockerfile ." } }
    stage('Push Image') { steps { sh 'aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY' ; sh 'docker push $ECR_REGISTRY/$IMAGE_NAME:latest' } }
    stage('Deploy') { steps { sh 'kubectl apply -f vproappdep.yml' } }
  }
}
```

Adjust this snippet to match your Jenkins agents and credential management.

## Configuration & Secrets

- Keep secrets outside source control. Use Kubernetes Secrets, SSM Parameter Store, or Vault for database passwords, API keys, and other sensitive values.
- Update `*-dep.yml` manifests to mount secrets or use environment variables from secret stores.

## Security & IAM

- Create an IAM role for your CI/CD service with permissions to push to ECR and update EKS resources (or use least-privilege service roles).
- For EKS nodes, ensure node IAM role has proper permissions for pulling images from ECR and accessing any managed services.

## Troubleshooting

- `ImagePullBackOff` often means the image URI or auth is wrong — verify ECR repo, tags and that the cluster/node IAM role can pull images.
- `CrashLoopBackOff` — check `kubectl logs <pod>` for stack traces or configuration errors.

## Recommended next steps

- Add a small architecture diagram (PNG/SVG) and link it from this README.
- Harden the Jenkins pipeline (add tests, artifact signing, and rollback strategies).
- Add GitHub Actions or other pipeline examples if you use non-Jenkins CI.

## Contributing

Contributions and improvements are welcome. Please open an issue for discussion or submit a pull request with clear descriptions of changes.

## Contact

Repo owner: `vhmedtaha` — open an issue or submit a pull request for feedback or collaboration.

<!-- README: polished to be actionable for developers and operators -->

# vproapp-devops-cicd

[![Build Status](https://img.shields.io/badge/build-unknown-lightgrey.svg)](https://example-ci.example.com) [![License](https://img.shields.io/badge/license-UNLICENSED-red.svg)](LICENSE)

Production-like Java web application example with containerized infrastructure, orchestration manifests, and CI/CD pipeline configuration.

## Table of Contents
- [Overview](#overview)
- [Quick Start](#quick-start)
- [Development Workflow](#development-workflow)
- [Repository Layout](#repository-layout)
- [Configuration & Environment](#configuration--environment)
- [Docker & Compose](#docker--compose)
- [CI/CD (Jenkins)](#cicd-jenkins)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [Maintainers & Contact](#maintainers--contact)

## Overview

- **Name:** `vproapp-devops-cicd`
- **Stack:** Java (Maven), Spring (MVC, Security), RabbitMQ, Elasticsearch, Memcached, Docker, Jenkins
- **Purpose:** Reference repo showing how to build/test/package a Spring web app, run supporting services in containers, and stitch everything together with CI/CD.

## Quick Start

Prerequisites:
- Java JDK 11+
- Maven 3.6+
- Docker Desktop (with Docker Compose)
- Git

Build the project and run unit tests:

```powershell
mvn clean package
mvn test
```

Start the full stack with Docker Compose:

```powershell
docker-compose up -d --build
```

Check logs:

```powershell
docker-compose logs -f
```

Stop and remove containers:

```powershell
docker-compose down
```

Notes:
- The repository includes sample compose manifests and service-specific deployment YAML files. Use them as a starting point and customize environment variables and resource limits for your environment.

## Development Workflow

- Create a feature branch: `git checkout -b feat/your-feature`
- Implement code under `src/main/java` and add tests to `src/test/java`
- Run `mvn test` locally and fix issues
- Build Docker image(s) and validate using `docker-compose`
- Push and open a PR for review

Good commit flow:

```text
git add .
git commit -m "feat: short description of change"
git push origin feat/your-feature
```

## Repository Layout

- `pom.xml` — Maven build and dependencies
- `Jenkinsfile` — CI pipeline definition
- `docker-compose.yml` — Default local orchestration
- `Docker-files/` — Docker build contexts for `app`, `db`, `web`
- `src/main/java/com/visualpathit/account` — application code (controllers, services, repos, utils)
- `src/main/resources` & `src/main/webapp/WEB-INF` — application properties and Spring XML configs
- `src/test/java` — unit and integration tests

## Configuration & Environment

- Primary app config: `src/main/resources/application.properties`
- Example secrets: `app-secret.yml` (do not store real secrets in repo)
- DB backup files: `Docker-files/db/db_backup.sql` and `src/main/resources/db_backup.sql`

Common environment variables (examples):

```text
DB_HOST=postgres
DB_PORT=5432
RABBITMQ_HOST=rabbitmq
ELASTICSEARCH_HOST=elasticsearch
MEMCACHED_HOST=memcached
SPRING_PROFILES_ACTIVE=dev
```

Tip: On Windows use PowerShell to export env vars for compose or use a `.env` file referenced by `docker-compose.yml`.

## Docker & Compose

Build the application image locally:

```powershell
docker build -t vproapp:local -f Docker-files/app/Dockerfile .
```

Start the whole stack (recommended for local integration testing):

```powershell
docker-compose up -d --build
```

Start a single service (example: only app):

```powershell
docker-compose up -d --build app
```

Useful commands:

```powershell
docker ps
docker-compose logs -f <service-name>
docker-compose down --volumes
```

Ports (typical examples):

- Application: `8080`
- RabbitMQ management: `15672`
- Elasticsearch: `9200`
- Memcached: `11211`

Adjust ports in compose files when they conflict with local services.

## CI/CD (Jenkins)

The included `Jenkinsfile` demonstrates a pipeline for building, testing and packaging the application and building Docker images. Typical stages:

- Checkout
- Build (`mvn -B -DskipTests=false clean package`)
- Unit tests (`mvn test`)
- Build Docker images
- Publish images to a registry (requires credentials)

Example Windows-aware stage (PowerShell agent) for Jenkins:

```groovy
stage('Build (Windows)') {
  steps {
    powershell 'mvn -B -DskipTests=false clean package'
  }
}
```

Configure Jenkins with credentials for your Docker registry and proper build agents (Linux or Windows) depending on your chosen image base.

## Testing

- Unit tests: `mvn test`
- Integration tests: add tests that start the compose stack (using Testcontainers or remote compose) and run against endpoints

Recommended: add a lightweight health-check endpoint and include a smoke-test stage in CI that verifies the app responds on `GET /health`.

## Troubleshooting

- If containers fail to start, run: `docker-compose logs -f` and inspect the failing service logs
- If Maven build fails, re-run with `-X` to enable debug output: `mvn -X clean package`
- On Windows, ensure Docker Desktop has enough memory/CPU and that WSL2 integration is enabled (if using WSL2)
- Port conflicts: change host ports in `docker-compose.yml` or stop the conflicting service

## Contributing

- Fork or branch from `main` and open a PR
- Add tests, keep changes small and well-documented
- Update documentation and `pom.xml` changes must include rationale

## Maintainers & Contact

- Use repository issues and pull requests for discussion. Provide team contact details or Slack channel here.

## Next Steps (suggested)

- Run `mvn test` and `docker-compose up -d --build` locally to validate the stack
- Configure Jenkins with registry credentials and test pipeline runs
- Add automated integration tests and a simple `/health` endpoint for CI smoke tests

---
_This README is tuned for clearer onboarding, reproducible local testing, and faster CI integration. If you'd like, I can:_

- run `mvn test` here and report failures
- create a Git commit and push a branch with this README update
- add real CI badge links and a maintainers block (provide URLs/contacts)


