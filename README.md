# CI/CD Pipeline for a Java Web Application (Jenkins → Maven → SonarQube → Nexus → Tomcat)

An end-to-end CI/CD pipeline that builds, tests, scans, stores, and deploys a Java web application — fully automated with Jenkins across a 3-VM infrastructure.

## Overview

Every push to this repository can be built and deployed through a single Jenkins pipeline that:

1. Pulls source code from GitHub
2. Builds and unit tests the app with Maven
3. Runs static code analysis with SonarQube
4. Packages the app into a WAR file
5. Publishes the WAR to a Nexus repository
6. Deploys the WAR to an Apache Tomcat server

## Architecture

```
GitHub (suffixscope/maven-web-app)
        │
        ▼
 VM1: Jenkins (Git, Maven, Docker)
        │
        ├─► Maven build & test
        │
        ▼
 VM1: SonarQube (Docker container) ── static analysis
        │
        ▼
 VM2: Nexus Repository ── stores built WAR
        │
        ▼
 VM3: Apache Tomcat 9.x ── runs the deployed app
```

## Infrastructure

| VM  | OS    | Tools Installed                          | Role |
|-----|-------|-------------------------------------------|------|
| VM1 | Ubuntu | Jenkins, Git, Maven, Docker (runs SonarQube) | CI orchestration & static analysis |
| VM2 | Linux  | Sonatype Nexus Repository                | Artifact storage |
| VM3 | Linux  | Apache Tomcat 9.x                        | Deployment target |

SonarQube runs as a Docker container on VM1 rather than on a dedicated VM, keeping the footprint to 3 VMs instead of 4.

## Tech Stack

- **CI/CD:** Jenkins (Declarative Pipeline)
- **Build:** Apache Maven
- **Code Quality:** SonarQube (Community Edition, Dockerized)
- **Artifact Repository:** Sonatype Nexus (`maven-releases`)
- **Deployment:** Apache Tomcat 9.x (`deploy` step, `tomcat9` adapter)
- **Project Coordinates:** `org.scopeindia:maven-web-app:1.0` (WAR packaging)

## Pipeline Stages

| # | Stage | Description |
|---|-------|-------------|
| 1 | Tool Install | Resolves JDK21 and Maven3 |
| 2 | Git Checkout | Clones `master` branch from GitHub |
| 3 | Maven Build | `mvn clean compile` |
| 4 | Unit Test | `mvn test` |
| 5 | SonarQube Scan | `mvn sonar:sonar` via `withSonarQubeEnv` |
| 6 | Package WAR | `mvn clean package` |
| 7 | Upload to Nexus | Publishes WAR via Nexus Artifact Uploader plugin |
| 8 | Deploy to Tomcat | Deploys WAR via Deploy-to-container plugin |

## Jenkins Configuration

**Plugins used:** Git, Pipeline, SonarQube Scanner for Jenkins, Nexus Artifact Uploader, Deploy to container.

**Global tools:** `JDK21`, `Maven3` (registered under *Manage Jenkins → Tools*).

**SonarQube server:** registered under *Manage Jenkins → System* with the exact name `SonarQube`, matching the `withSonarQubeEnv('SonarQube')` reference in the Jenkinsfile.

**Credentials:**
| ID | Purpose |
|----|---------|
| `nexus-creds` | Nexus username/password |
| `tomcat-creds` | Tomcat manager username/password |

## Getting Started

1. Provision 3 VMs and open ports `8080` (Jenkins/Tomcat), `9000` (SonarQube), `8081` (Nexus) between them.
2. Install Java JDK 21, Jenkins, Git, and Maven on VM1.
3. Run SonarQube on VM1 via Docker:
   ```bash
   docker run -d --name sonarqube -p 9000:9000 sonarqube:community
   ```
4. Install Nexus Repository on VM2 and Apache Tomcat 9.x on VM3.
5. In Jenkins, install the required plugins and configure the tools, SonarQube server, and credentials listed above.
6. Create a Pipeline job pointing at this repository with script path `Jenkinsfile`.
7. Run the build.

## Verified Result

The pipeline completed with `Finished: SUCCESS` across all 8 stages. The WAR was published to Nexus and deployed live to Tomcat.

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| Groovy syntax error at Nexus/Tomcat stages | Raw IP:port literals wrapped in `${...}` string interpolation | Used pre-declared environment variables instead |
| SonarQube installation not found | Server name not registered under *Manage Jenkins → System* | Registered a server named exactly `SonarQube` |
| 401 Unauthorized on Nexus upload | Incorrect Nexus credentials / missing deploy permission | Corrected `nexus-creds` and confirmed deploy rights on `maven-releases` |
| VM IPs changed between runs | Cloud instances with dynamic public IPs | Updated all Jenkinsfile environment variables before the final run |

## References

- [Jenkins](https://www.jenkins.io/download/) · [Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Apache Maven](https://maven.apache.org/download.cgi)
- [SonarQube (Docker)](https://hub.docker.com/_/sonarqube)
- [Sonatype Nexus](https://help.sonatype.com/en/download.html)
- [Apache Tomcat](https://tomcat.apache.org/download-10.cgi)
- [Nexus Artifact Uploader Plugin](https://plugins.jenkins.io/nexus-artifact-uploader/)
- [Deploy to Container Plugin](https://plugins.jenkins.io/deploy/)

---
*Internship project — Scope India, Thiruvananthapuram (Jan – Jul 2026)*
