# Workshop 2 – Groupe 1
Feddouli Wiam - Metmari Hafsa - Lboudaoudi Zaynab - Aalaoui Ismail

---

# DevOps Pipeline – CI/CD & Code Quality Analysis

This repository documents **two complete CI approaches** used to build, test, and perform **static code quality analysis** on a Java Maven project using **SonarQube**.

The goal of this project was to understand and compare:

* A **fully managed CI pipeline using GitHub Actions**
* A **self-hosted CI pipeline using Jenkins + Docker**

Both approaches analyze the same Java Maven project and publish **code quality metrics** (bugs, vulnerabilities, code smells, coverage, duplications) to **SonarQube**.

---

## Technologies Used

* Java 17
* Maven
* SonarQube (Dockerized)
* GitHub Actions
* Jenkins
* Docker & Docker volumes
* Ngrok (for exposing local SonarQube to Jenkins)

---

# Method 1: GitHub Actions + SonarQube

## Overview

In this approach, I used **GitHub Actions as a CI pipeline** to:

1. Automatically **build and test** the Maven project
2. Run **static code analysis**
3. Send the analysis results to **SonarQube**

This process is triggered **on every push to the `main` branch**.

---

## SonarQube Setup

* SonarQube was deployed locally using **Docker Desktop**
* From the SonarQube UI:

  * A **user token** was generated (`Account → Security`)
  * This token allows automated tools to authenticate securely

### GitHub Secrets Configuration

To allow GitHub Actions to communicate securely with SonarQube, the following **repository secrets** were added:

* `SONAR_HOST_URL` → URL of the SonarQube server
* `SONAR_TOKEN` → Authentication token generated from SonarQube

Secrets are encrypted and never exposed in logs or source code.

---

## GitHub Actions Workflow (build.yaml) – Explanation

The workflow defines a **CI pipeline** composed of these steps:

### 1. Source Code Checkout

* Retrieves the full Git history
* Required by SonarQube for accurate issue tracking and blame information

### 2. Java Environment Setup

* Installs **JDK 17** on the GitHub runner
* Ensures compatibility with the Maven project

### 3. Dependency Caching

Two caches are configured to improve performance:

* **SonarQube cache** → stores Sonar analyzer components
* **Maven cache** → stores downloaded dependencies (`~/.m2`)

This significantly reduces build time on subsequent runs.

### 4. Build & Static Analysis

* Maven runs the full lifecycle (`verify` phase)
* The **Sonar Maven Plugin** sends:

  * Source code
  * Bytecode
  * Test results
  * Coverage data

  to SonarQube for **quality analysis and visualization**.

---

## Result

* Analysis results are available in the **SonarQube dashboard**
* GitHub Actions provides:

  * Fully automated CI
  * No infrastructure maintenance
  * Easy integration with GitHub repositories

---

# Method 2: Jenkins + Docker + SonarQube

## Overview

In this approach, I built a **self-hosted Jenkins CI pipeline** using:

* Jenkins running inside Docker
* Custom Docker images for build isolation
* SonarQube for static analysis

This method provides **full control** over tools, environments, and execution.

---

## Jenkins Architecture

Jenkins is used to:

1. Execute a scripted pipeline job (defined directly in Jenkins)
2. Build and test the project using Maven
3. Run **SonarScanner**
4. Send analysis results to SonarQube

All build steps are executed inside **Docker containers** to ensure consistency.

---

## Jenkins Pipeline Script

This Jenkins pipeline runs inside a Docker container using the custom image my-maven-git-sonarscanner:latest, which includes Maven, Git, and SonarScanner tools.

* **Checkout stage:** Cleans the workspace and clones the project repository from GitHub.
* **Build stage:** Navigates to the Maven project directory and runs mvn clean test package to build the project and execute tests.
* **SonarQube Analysis stage:** Runs SonarScanner with Jenkins-managed credentials to analyze the code and send the results to the configured SonarQube server. The analysis uses project-specific keys and paths to source code and compiled classes.

This pipeline ensures the entire build and code quality analysis process is containerized, reproducible, and integrated with SonarQube for continuous quality monitoring.

---

## Custom Maven + Git + SonarScanner Image

This image is used as the **Jenkins build agent**.

### Purpose of Each Tool

* **Maven** → Builds the Java project and runs tests
* **Git** → Clones the repository inside the container
* **SonarScanner** → Sends source code and metrics to SonarQube

### Dockerfile: `my-maven-git-sonarscanner`

```dockerfile
FROM maven:4.0.0-rc-5-amazoncorretto-25-debian-trixie

# Install Git
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*

# Install SonarScanner
ENV SONAR_SCANNER_VERSION=5.0.1.3006

RUN apt-get update && apt-get install -y curl unzip && \
    curl -L -o /tmp/sonar.zip \
    "https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${SONAR_SCANNER_VERSION}-linux.zip" \
    && unzip /tmp/sonar.zip -d /opt \
    && mv /opt/sonar-scanner-* /opt/sonar-scanner \
    && rm /tmp/sonar.zip

ENV PATH="$PATH:/opt/sonar-scanner/bin"
```

This image guarantees:

* Reproducible builds
* Identical tooling across all executions
* Isolation from the Jenkins host system

---

## Jenkins Server with Docker Support

Jenkins itself runs inside a Docker container.

### Dockerfile: `jenkins-docker-git`

```dockerfile
FROM jenkins/jenkins:latest

USER root

# Install Docker CLI
RUN apt-get update && \
    apt-get install -y docker.io && \
    apt-get clean

USER jenkins
```

### Why Docker Inside Jenkins?

* Allows Jenkins to run **Docker-based agents**
* Enables building images and running containers from pipelines
* Provides flexibility without installing Docker on the host system

---

## Jenkins Container Execution

Jenkins is started using:

```bash
docker run -d \
  --name jenkins \
  -p 9090:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --user root \
  jenkins-docker
```

### Command Explanation

* `-d` → Run in background
* `-p 9090:8080` → Access Jenkins UI
* `-p 50000:50000` → Jenkins agents communication
* `jenkins_home` volume → Persistent configuration & jobs
* Docker socket mount → Allows Jenkins to control Docker
* `--user root` → Required to access Docker daemon

---

## Jenkins Plugins Used

* **Docker Plugin** → Run build agents as containers
* **Docker Pipeline Plugin** → Define Docker steps in pipelines
* **SonarQube Scanner Plugin** → Integrates SonarQube with Jenkins

These plugins allow Jenkins to orchestrate containerized builds and analysis.

---

## SonarQube Integration with Jenkins

### Why Ngrok Is Required

SonarQube runs **locally inside Docker**, while Jenkins needs a reachable HTTP endpoint.

* SonarQube does not accept `localhost` from containerized Jenkins
* **Ngrok** exposes the local SonarQube server to a public HTTP URL

This allows Jenkins to securely communicate with SonarQube.

---

### SonarQube Server Configuration in Jenkins

Configured via:

`Manage Jenkins → System → SonarQube Servers`

* Server URL → Ngrok-generated HTTP URL
* Authentication → Jenkins credential

#### Jenkins Credential

* Type: **Secret Text**
* Secret: SonarQube token
* ID: Custom identifier used by pipelines

---

### SonarQube Scanner Installation

Configured via:

`Manage Jenkins → Tools → SonarQube Scanner`

* Registers the scanner used during analysis
* Referenced by name inside the pipeline

---

## Result

* Jenkins executes a **containerized Maven build**
* SonarScanner sends metrics to SonarQube
* Results are visualized in the SonarQube dashboard

This method demonstrates:

* Full CI control
* Real-world enterprise setup
* Deep understanding of DevOps tooling

---

## Screenshots

 * The following screenshot shows a successful Jenkins job where the analysis data has been sent to SonarQube:

<img width="1232" height="639" alt="image" src="https://github.com/user-attachments/assets/96bc94a8-4c4e-48b5-9703-ca1100484d3d" />


* This screenshot displays the history of code analyses triggered by our CI pipeline in SonarQube:

<img width="1564" height="802" alt="image" src="https://github.com/user-attachments/assets/23cd94fe-7cc8-463e-a19a-65a0d0a4546a" />

---

## Conclusion

This project demonstrates two professional CI strategies:

| GitHub Actions  | Jenkins          |
| --------------- | ---------------- |
| Fully managed   | Self-hosted      |
| Minimal setup   | Maximum control  |
| Fast onboarding | Enterprise-grade |

Both successfully implement **Continuous Integration + Static Code Quality Analysis** using SonarQube.

---

> This repository focuses on **CI and code quality**, not deployment.
