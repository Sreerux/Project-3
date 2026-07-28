# CI/CD Deployment of a Spring Boot REST API on Amazon EKS with RDS MySQL

A complete DevOps pipeline that builds a Spring Boot REST API, containerizes it, and deploys it to Amazon EKS — connected to a managed Amazon RDS MySQL database and exposed through an AWS Load Balancer.

## Overview

Every push to GitHub triggers a Jenkins pipeline that:

1. Pulls the latest source code
2. Builds the Spring Boot app with Maven
3. Builds and pushes a Docker image to Docker Hub
4. Updates the Kubernetes deployment on Amazon EKS
5. Connects the running app to Amazon RDS MySQL
6. Serves users through an AWS Load Balancer

## Architecture

```
Developer → GitHub → Jenkins CI
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
     Maven Build   Docker Build   Docker Push
                                       │
                                       ▼
                                  Docker Hub
                                       │
                                       ▼
                                 kubectl apply
                                       │
                                       ▼
                              Amazon EKS Cluster
                          ┌────────────┴────────────┐
                          ▼                          ▼
                  Spring Boot Pod            Spring Boot Pod
                          │
                          ▼
                   Amazon RDS MySQL
                          │
                          ▼
               AWS Load Balancer (ELB) → Users
```

## Tech Stack

| Category | Tool |
|----------|------|
| Cloud | AWS |
| Source Control | GitHub |
| CI/CD | Jenkins |
| Build Tool | Maven |
| Containerization | Docker |
| Container Registry | Docker Hub |
| Orchestration | Amazon EKS |
| Database | Amazon RDS (MySQL) |
| CLI | kubectl |
| OS | Ubuntu Linux |

**AWS services used:** IAM, EC2, EKS, RDS, VPC, Security Groups, Load Balancer.

## Infrastructure

| Instance | Type | Role |
|----------|------|------|
| `jenkins-sever` | c7i-flex.large | Jenkins CI/CD server |
| `mycluster-clus...` (x2) | c7i-flex.large | EKS worker nodes |

## Pipeline Stages

| # | Stage | Description |
|---|-------|-------------|
| 1 | Tool Install | Resolves JDK21, maven3 |
| 2 | Checkout | Clones `main` branch, workspace cleaned first |
| 3 | Verify Source | Confirms repo contents |
| 4 | Debug Dockerfile | Prints Dockerfile + full repo tree for sanity check |
| 5 | Build Application | `mvn clean package -DskipTests` |
| 6 | Build & Push Docker Image | Builds, tags, and pushes image to Docker Hub |
| 7 | Deploy to EKS | Updates kubeconfig, applies manifest, restarts + verifies rollout |
| 8 | Verify Deployment | Checks deployments, pods, and service |

## Kubernetes Resources

- **Deployment** `products-api` — 2 replicas, image `sreerx/products-api:latest`, container port `8082`
- **Service** `products-api-service` — type `LoadBalancer`, port `80` → target port `8082`

## Database

The Spring Boot app connects to Amazon RDS MySQL via JDBC at:
```
products-db.cjyu66i2w9gl.ap-south-1.rds.amazonaws.com
```
Storing data in RDS (instead of inside the cluster) means data survives pod restarts, rescheduling, and redeployments.

## Jenkins Configuration

**Credentials:**
| ID | Purpose |
|----|---------|
| `dockerhub-creds` | Docker Hub username/password, injected as `$DOCKER_USER` / `$DOCKER_PASS` |
| AWS credentials | Used by `aws eks update-kubeconfig` to authenticate against the EKS cluster |

**Environment variables (Jenkinsfile):**
```groovy
IMAGE_NAME  = "sreerx/products-api"
IMAGE_TAG   = "${BUILD_NUMBER}"
AWS_REGION  = "ap-south-1"
CLUSTER_NAME = "mycluster-cluster4"
```

## Getting Started

1. Provision a Jenkins EC2 instance with Git, Maven, Docker, and AWS CLI.
2. Create/confirm the EKS cluster (`mycluster-cluster4`, `ap-south-1`) and an RDS MySQL instance, with the RDS security group allowing inbound access from EKS worker nodes.
3. Add `dockerhub-creds` in Jenkins and ensure AWS credentials are available to the pipeline.
4. Ensure the repo root contains `Jenkinsfile`, `Dockerfile`, `pom.xml`, and `Deployment.yml`.
5. Create a Jenkins Pipeline job pointing at this repository.
6. Run the build.

## Repository Structure

```
products-api/
├── Jenkinsfile
├── Dockerfile
├── pom.xml
├── README.md
├── kubernetes/
│   ├── deployment.yaml
│   └── service.yaml
├── sql/
│   └── DB_Setup.sql
├── screenshots/
└── src/
```

## Verified Result

Build #8 finished in ~41 seconds with all 9 stages passing. Confirmed live via:
```
http://<elb-address>.ap-south-1.elb.amazonaws.com/api/products/11
```
returning the expected product record read from RDS.

## Troubleshooting Guide

| Issue | Cause | Resolution |
|-------|-------|------------|
| Docker permission denied | Jenkins user not in Docker group | Added Jenkins to Docker group |
| Maven build failed | Missing dependencies | Verified `pom.xml` and rebuilt |
| Image push failed | Incorrect Docker credentials | Updated Docker Hub credentials in Jenkins |
| EKS authentication failed | kubeconfig not updated | Ran `aws eks update-kubeconfig` |
| Pods not starting | Wrong image or configuration | Checked pod logs, corrected deployment |
| Application inaccessible | Incorrect container port | Fixed Deployment/Service port config |
| LoadBalancer pending | AWS resource creation delay | Waited for ELB provisioning |
| Empty reply from server | Port mismatch (8081 vs 8082) | Aligned Deployment/Service to port 8082 |
| Database connection issues | Security Group or JDBC URL | Allowed EKS access to RDS, verified config |
| Deployment not updating | Old image cached | Used `kubectl rollout restart deployment/products-api` |

## References

- [Jenkins](https://jenkins.io/download) · [Pipeline Syntax](https://jenkins.io/doc/book/pipeline/syntax)
- [Apache Maven](https://maven.apache.org/download.cgi)
- [Docker Engine](https://docs.docker.com/engine/install)
- [Docker Hub](https://hub.docker.com)
- [Amazon EKS](https://aws.amazon.com/eks)
- [Amazon RDS for MySQL](https://aws.amazon.com/rds/mysql)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [AWS CLI](https://aws.amazon.com/cli)

---
*Internship project — Scope India, Thiruvananthapuram (Jan – Jul 2026)*
