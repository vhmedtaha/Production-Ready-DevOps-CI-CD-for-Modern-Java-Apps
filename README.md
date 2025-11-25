# vproapp-devops-cicd

> A hands‑on, CI/CD-ready sample for a Java web application demonstrating containerized build & deployment with Docker, Docker Compose, Jenkins and supporting services (RabbitMQ, DB, Elasticsearch).

## Overview

This repository contains a complete example of packaging a Java web application into containers and automating delivery with a Jenkins pipeline. The goal is to provide reusable patterns for developers and DevOps engineers who want a pragmatic starting point for CI/CD with Java webapps.

Key components:

- Dockerfiles for `app`, `db`, and `web` (under `Docker-files/`).
- `docker-compose.yml` for local multi-container orchestration.
- Kubernetes-style YAMLs and service manifests (`*-CIP.yml`, `*-dep.yml`) for deploying services like RabbitMQ and database.
- `Jenkinsfile` demonstrating a pipeline for build, test, image build, and deployment steps.
- A sample Java web application in `src/` (controllers, services, utils, webapp views).

## Prerequisites

- Java JDK (compatible with project build configuration)
- Maven (for building the Java app)
- Docker Engine
- Docker Compose (optional, for local compose runs)
- Jenkins (if running CI pipeline)

## Quickstart — Run locally with Docker Compose

1. Build the application jar (from repo root):

```powershell
mvn clean package -DskipTests
```

2. Build images (optional — the `Dockerfiles` are provided):

```powershell
docker build -t vproapp:local -f Docker-files/app/Dockerfile .
docker build -t vproweb:local -f Docker-files/web/Dockerfile .
```

3. Start services with Docker Compose:

```powershell
docker-compose up --build
```

This will start the app, database and supporting services according to `docker-compose.yml`. Visit the web UI at the address shown in the compose output (typically `http://localhost:8080` depending on the web container configuration).

## CI/CD (Jenkins)

The `Jenkinsfile` in repository root shows a canonical pipeline that:

- Checks out source code
- Runs Maven build and unit tests
- Builds Docker images
- Pushes images to a registry (configure credentials in your Jenkins)
- Applies deployment manifests to the target environment (example manifests included)

To adapt the pipeline:

- Customize the Docker registry and credentials step.
- Adjust build/test commands to match your Java/Maven settings.
- Replace deployment steps with your environment's deployment strategy (kubectl, helm, or remote docker-compose deploy).

## Repository structure

Highlights of the repo layout:

- `Docker-files/` — `app/`, `db/`, `web/` Dockerfiles and nginx config
- `docker-compose.yml` — Local compose orchestration
- `Jenkinsfile` — Example pipeline for CI/CD
- `*.yml` (`*-CIP.yml`, `*-dep.yml`) — Service and deployment manifests (RabbitMQ, DB, etc.)
- `src/` — Java web application source, resources and tests

## How to customize

- Swap the Maven/Java versions in the `Dockerfile` and Maven configuration.
- Update environment variables in `docker-compose.yml` and the provided YAMLs for your infrastructure (DB credentials, hostnames, ports).
- Add or remove services (e.g., Elasticsearch, Memcached) by editing the compose file and manifests.

## Suggested improvements / next steps

- Add a small architecture diagram (PNG/SVG) that shows CI pipeline and runtime topology.
- Configure automated integration tests in the Jenkins pipeline.
- Publish Docker images to a registry (Docker Hub, GitHub Container Registry, or private registry).

## Contributing

Contributions are welcome. For changes to CI/CD files or Docker images, please ensure builds run locally (via Maven and Docker) and update the `Jenkinsfile` or compose manifests as needed.

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


