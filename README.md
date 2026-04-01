# Jenkins CI/CD Pipeline — Spring PetClinic Deployment

> **This repository is a fork of [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) used to demonstrate a complete Jenkins CI/CD pipeline with multi-node AWS infrastructure.**

## Project Overview

A fully automated CI/CD pipeline that builds, tests, scans, and deploys the Spring PetClinic application using Jenkins Multibranch Pipeline across 3 AWS EC2 instances.

**Repository:** [sushanthreddy07/jenkins-maven](https://github.com/sushanthreddy07/jenkins-maven)  
**Author:** Sushanth Reddy Rachala ([@sushanthreddy07](https://github.com/sushanthreddy07))

---

## Architecture

```
┌─────────────┐         ┌──────────────────┐
│  Developer   │  push   │     GitHub       │
│  (Local)     │────────▶│  jenkins-maven   │
└─────────────┘         │  main + develop  │
                        └────────┬─────────┘
                                 │ webhook (instant trigger)
                                 ▼
                        ┌──────────────────┐
                        │  Jenkins Master   │
                        │  EC2: t2.medium   │
                        │  Ubuntu 22.04     │
                        │  + SonarQube :9000│
                        └────────┬─────────┘
                                 │ SSH (RSA keys)
                                 ▼
                        ┌──────────────────┐
                        │  Jenkins Agent    │
                        │  (slave)          │
                        │  EC2: t2.micro    │
                        │  Ubuntu 24.04     │
                        │  Java, Maven,Trivy│
                        └────────┬─────────┘
                                 │ SCP + SSH (main branch only)
                                 ▼
                        ┌──────────────────┐
                        │  App Server       │
                        │  EC2: t2.micro    │
                        │  Ubuntu 24.04     │
                        │  App runs on :8080│
                        └──────────────────┘
```

---

## Pipeline Stages

```
Checkout ──▶ Build & Test ──▶ SonarQube ──▶ Trivy Scan ──▶ Package ──▶ Deploy
   │              │               │              │             │           │
   │         mvn clean test   Code Quality   Security     mvn package   SCP + SSH
   │          57 tests        17 issues      0 CRITICAL    .jar file   (main only)
   ▼              ▼               ▼              ▼             ▼           ▼
  Git SCM       BUILD           SONARQUBE      TRIVY       ARTIFACT    APP SERVER
               SUCCESS          PASSED         PASSED      ARCHIVED      :8080
```

| Stage | Tool | What It Does |
|-------|------|--------------|
| Checkout Code | Git SCM | Pulls latest code from GitHub |
| Build & Test | Maven (`mvn clean test`) | Compiles code and runs 57 unit tests |
| SonarQube Analysis | SonarQube + Maven | Static code quality and maintainability analysis |
| Security Scan | Trivy v0.69.3 | Scans dependencies for CRITICAL vulnerabilities |
| Package | Maven (`mvn package`) | Builds `.jar` artifact and archives in Jenkins |
| Deploy | SSH/SCP | Copies `.jar` to App Server and starts it (main branch only) |

---

## Branch Strategy

| Branch | Stages Executed | Deploys To |
|--------|----------------|------------|
| `main` | All 6 stages including Deploy | Production (App Server :8080) |
| `develop` | First 5 stages (Deploy skipped) | — |

---

## Tech Stack

| Category | Technology |
|----------|-----------|
| Application | Spring PetClinic (Java 17, Maven, Spring Boot 4.0.3) |
| CI/CD | Jenkins 2.541.3 (Multibranch Pipeline) |
| Code Quality | SonarQube (Docker container) |
| Security Scanning | Trivy v0.69.3 |
| Infrastructure | AWS EC2 (3 instances) |
| OS | Ubuntu 22.04 / 24.04 LTS |
| Notifications | Gmail SMTP (TLS, port 587) |
| SCM Trigger | GitHub Webhook |

---

## Infrastructure Details

| Instance | Type | Role | Software Installed |
|----------|------|------|--------------------|
| Jenkins Master | t2.medium (4GB RAM) | Pipeline orchestration | Jenkins, Docker, SonarQube |
| Jenkins Agent (slave) | t2.micro (1GB + 2GB swap) | Build execution | Java 17, Maven, Trivy |
| App Server | t2.micro | Application hosting | Java 17 |

### Connectivity
- All instances in the **same VPC**, communicating via **private IPs**
- SSH authentication via **RSA 4096-bit key pairs** generated on Jenkins Master (`jenkins` user)
- Public keys distributed to Agent and App Server via `authorized_keys`

---

## Jenkins Credentials

| ID | Type | Purpose |
|----|------|---------|
| `github-cred` | Username + PAT | GitHub repository access |
| `app-server-ssh` | SSH Private Key | Deploy artifacts to App Server |
| `slave` | SSH Private Key | Connect Jenkins Agent |
| `sonarqube-token` | Secret Text | SonarQube API authentication |

---

## Jenkins Plugins Used

- Pipeline & Pipeline: Multibranch
- Git
- Credentials Binding
- SSH Agent & SSH Build Agents
- SonarQube Scanner
- Email Extension

---

## Build Results

| Metric | Value |
|--------|-------|
| Total Tests | 57 |
| Passed | 55 |
| Skipped | 2 (Docker-based integration tests) |
| Failed | 0 |
| CRITICAL Vulnerabilities | 0 |
| HIGH Vulnerabilities | 2 (jackson-core — not blocking) |
| SonarQube Issues | 17 (3 Blocker, 3 High, 9 Medium, 2 Low) |
| Build Time | ~40 seconds |

---

## Extra Features Implemented

### 1. GitHub Webhook
Instantly triggers Jenkins on every `git push` — no polling required. Configured at GitHub repo → Settings → Webhooks.

### 2. Email Notifications
Sends success/failure emails via Gmail SMTP after every build.

### 3. SonarQube Code Quality
Runs static analysis via SonarQube (Docker on Master). Identified 17 maintainability issues with an estimated 1h 51min fix effort.

### 4. Jenkins Users
Created additional users with role-based access control.

---

## Issues Encountered & Resolved

| Issue | Cause | Resolution |
|-------|-------|------------|
| Agent SSH rejected private key | Credential username `slave` instead of `ubuntu` | Changed username to `ubuntu` |
| Agent going offline during builds | t2.micro has only 1GB RAM | Added 2GB swap space |
| Trivy failing pipeline on HIGH vulns | `--severity HIGH,CRITICAL` too strict | Changed to `--severity CRITICAL` |
| `sshagent` DSL method not found | SSH Agent plugin missing | Installed SSH Agent plugin |
| SonarQube unreachable from Agent | URL was `localhost:9000` | Changed to Master's private IP |
| Email SMTP connection error | SSL on port 465 not working | Switched to TLS on port 587 |
| Pipeline stuck waiting for executor | Label `maven-agent` vs node label `slave` | Updated Jenkinsfile to `slave` |

---

## How to Use

### Automatic (Webhook)
```bash
git add .
git commit -m "your changes"
git push origin main
```
Jenkins auto-triggers within seconds via webhook.

### Manual
Jenkins Dashboard → `jenkins-cicd-pipeline` → select branch → **Build Now**

### Access the App
After a successful `main` branch build:
```
http://<APP_SERVER_PUBLIC_IP>:8080
```

---

## Project Structure

```
spring-petclinic/
├── Jenkinsfile              # CI/CD pipeline definition
├── pom.xml                  # Maven build configuration
├── src/
│   ├── main/
│   │   ├── java/            # Application source code
│   │   └── resources/       # Config, templates, static assets
│   └── test/
│       └── java/            # Unit and integration tests
├── mvnw                     # Maven wrapper
└── README.md                # This file
```

---

# About Spring PetClinic

The application used in this CI/CD pipeline is [Spring PetClinic](https://github.com/spring-projects/spring-petclinic), a sample Spring Boot application.

## Run Locally

```bash
git clone https://github.com/sushanthreddy07/jenkins-maven.git
cd jenkins-maven
./mvnw spring-boot:run
```

Access at [http://localhost:8080](http://localhost:8080).

<img width="1042" alt="petclinic-screenshot" src="https://cloud.githubusercontent.com/assets/838318/19727082/2aee6d6c-9b8e-11e6-81fe-e889a5ddfded.png">

## Database Configuration

By default, Petclinic uses an in-memory H2 database. For persistent storage:

```bash
# MySQL
docker run -e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=petclinic -p 3306:3306 mysql:9.6

# PostgreSQL
docker run -e POSTGRES_USER=petclinic -e POSTGRES_PASSWORD=petclinic -e POSTGRES_DB=petclinic -p 5432:5432 postgres:18.3
```

Set profile: `spring.profiles.active=mysql` or `spring.profiles.active=postgres`

## Prerequisites

- Java 17 or newer (full JDK)
- [Git](https://help.github.com/articles/set-up-git)
- Maven (included via `mvnw` wrapper)

## Building a Container

```bash
./mvnw spring-boot:build-image
```

## License

The Spring PetClinic sample application is released under version 2.0 of the [Apache License](https://www.apache.org/licenses/LICENSE-2.0).
