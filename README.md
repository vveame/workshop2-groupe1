# Workshop 3 – Groupe 1
Feddouli Wiam - Metmari Hafsa - Lboudaoudi Zaynab - Aalaoui Ismail

---

# DevOps Pipeline – CI/CD & Code Quality Analysis

This repository documents **two complete CI approaches** used to build, test, and perform **static code quality analysis** on a Java Maven project using **SonarQube**.

The goal of this project was to understand and compare:

* A **fully managed CI pipeline using GitHub Actions**
* A **self-hosted CI pipeline using Jenkins + Docker**, extended with **security analysis** (UPDATED)

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
* GitHub Dependabot
* OWASP Dependency-Check
* Trivy (NEW)

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

# Method 2: Jenkins + Docker + SonarQube (UPDATED)

## Overview

In this approach, I built a **self-hosted Jenkins CI pipeline** using:

* Jenkins running inside Docker
* Custom Docker images for build isolation
* SonarQube for static analysis
* **OWASP Dependency-Check for dependency vulnerability scanning**
* Trivy for container image scanning (NEW)

This method provides **full control** over tools, environments, and execution.

---

## Jenkins Architecture

Jenkins is used to execute a scripted pipeline job (defined directly in Jenkins) that :

1.Scans Docker image vulnerabilities with Trivy (NEW)
2. Clone the GitHub repository
3. Build and test the Maven project
4. Perform **OWASP dependency security analysis**
5. Run SonarQube analysis
6. Publish reports and quality metrics

All build steps are executed inside **Docker containers** to ensure consistency.

---

## Security Enhancements Added (NEW)

In addition to build and static code quality analysis, this method was extended to include dependency security monitoring using GitHub Dependabot and OWASP Dependency-Check.
Trivy is also integrated to scan the custom build image for vulnerabilities:
These tools strengthen the pipeline by detecting outdated libraries and known vulnerabilities (CVEs) early in the CI process.

---

### Dependabot – Automated Dependency Updates

Dependabot was enabled to **automatically detect and update vulnerable dependencies**.

#### Configuration Steps
- Enabled in:  
  `Repository Settings → Advanced Security`
  - Dependabot alerts
  - Dependabot security updates
- Added configuration file:  
  `.github/dependabot.yml`
- Dependabot works **only on the default branch**
- For forked repositories, Dependabot must be enabled in:  
  `Insights → Dependency graph → Dependabot`

#### Behavior
- Dependabot automatically detects vulnerable dependencies
- Creates a **pull request** with the proposed update
- The user can:
  - Review and merge the PR
  - Or close it if not needed

#### dependabot.yaml - Explanation

This Dependabot configuration enables automated monitoring and updating of Maven dependencies for the project.

Dependabot checks the pom.xml file located in the /maven directory and runs weekly every Thursday at 11:18 (Africa/Casablanca timezone).
It scans both direct and transitive dependencies and automatically opens up to 5 pull requests when updates are available.

Each pull request uses a standardized commit message starting with deps and includes the scope of the change. Pull requests are automatically assigned and requested for review by the specified user.
Related dependencies (such as Spring libraries) are grouped into a single pull request to reduce noise and simplify maintenance.
 
---

### OWASP Dependency-Check

OWASP Dependency-Check analyzes third-party dependencies and detects known vulnerabilities using:
* NVD (National Vulnerability Database)
* GitHub Advisory Database
* CISA Known Exploited Vulnerabilities Catalog

To enhance security analysis in Jenkins:

- The **OWASP Dependency-Check plugin** was added to `pom.xml`
- This plugin scans project dependencies against known CVEs
- It is automatically executed during the Maven `verify` phase

---

### Trivy – Container Image Scanning (NEW)

**Trivy** is a comprehensive vulnerability scanner for containers and other artifacts that detects security issues in OS packages and application dependencies.

In this pipeline, **Trivy scans the custom Docker image (`my-maven-git-sonarscanner:latest`) before any build stage**, ensuring that vulnerabilities in the base image or installed tools (Maven, Git, SonarScanner) don't compromise the pipeline. It enforces a security policy where:
- **CRITICAL** vulnerabilities fail the build immediately
- **MEDIUM** vulnerabilities mark the build as unstable
- **LOW** vulnerabilities are logged for awareness

This adds an **essential security gate** that prevents using compromised build environments in the CI/CD process.

#### Custom Trivy image with Caching

**The custom `trivy-cached` image pre-downloads vulnerability databases** so Trivy doesn't need to fetch them on every scan. This significantly **reduces scan time** from minutes to seconds in the pipeline.

```Dockerfile
# Dockerfile for trivy-cached
FROM aquasec/trivy:latest

# Pre-download and cache vulnerability databases
RUN trivy image --download-db-only && \
    trivy image --download-java-db-only

# Set cache directory
VOLUME /root/.cache/trivy
```

Without caching, each Trivy scan would:
1. Download 100+ MB of vulnerability data from remote servers
2. Take 2-5 minutes per scan
3. Consume more bandwidth and pipeline time

With the cached image:
1. Vulnerability databases are pre-loaded in the image
2. Scans complete in 10-30 seconds
3. Consistent performance regardless of network conditions
4. Reduced external dependencies during pipeline execution

This optimization is **critical for CI/CD** where fast feedback loops are essential, and it ensures the security scan doesn't become a bottleneck in the pipeline.

---

## Jenkins Pipeline (Jenkinsfile)

### Script Explanation

This enhanced Jenkins pipeline implements a **multi-stage security-first CI/CD workflow** using Docker containers for isolation and reproducibility. The pipeline features parameterized execution, container security scanning, and comprehensive quality gates.

* **Pipeline Configuration:** Uses `agent none` for optimized resource management, with parameters allowing conditional OWASP dependency scanning.
* **Trivy Scan Stage:** Performs container image vulnerability scanning on the custom `my-maven-git-sonarscanner:latest` image using a pre-cached Trivy image. A complete JSON vulnerability report is archived.
* **Checkout Stage:** Cleans the workspace and clones the project repository from GitHub into a fresh Docker container with persistent Maven repository caching.
* **Build & Test Stage:** Compiles the Java Maven project, runs unit tests, and archives test results and reports using the JUnit plugin for test result tracking.
* **OWASP Dependency Check Stage:** Conditionally executes based on pipeline parameters, scanning project dependencies for known vulnerabilities and archiving HTML security reports.
* **SonarQube Analysis Stage:** Performs static code analysis using SonarScanner, sending code quality metrics to the configured SonarQube server with Jenkins-managed authentication.
* **Post-build Actions:** Provides clear status messages indicating pipeline success, warnings, or failure based on the security and quality gates.

This enhanced pipeline ensures **security scanning happens before code compilation**, implements **flexible security controls**, maintains **clean workspace isolation per stage**, and provides **comprehensive artifact archiving** for all test and security reports.

### Workspace Separation & Container Strategy

**Workspace separation** ensures each stage runs in its own clean environment to prevent cross-stage contamination. The **Maven stages** execute inside the `my-maven-git-sonarscanner` container because they need Maven, Java, and project dependencies. **Trivy scans outside this container** because it needs direct Docker daemon access to scan the image itself, not run project code. This separation maintains security (Trivy examines the build environment) while keeping build stages isolated and reproducible within their dedicated containers.

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


* This screenshot shows an automatically generated Dependabot pull request detected on the default branch:
![WhatsApp Image 2025-12-18 at 1 03 46 PM](https://github.com/user-attachments/assets/3793d280-3704-4124-be15-e43613f6e4b1)


* This screenshot shows a successful Jenkins pipeline execution with archived OWASP Dependency-Check reports generated during the build and test stage:
![WhatsApp Image 2025-12-19 at 8 59 31 PM](https://github.com/user-attachments/assets/4c4adcf6-3ef4-466e-bfb5-91ac4502eb9a)


* This screenshot displays the OWASP Dependency-Check HTML report, summarizing the vulnerability analysis of project dependencies using public security databases:
![WhatsApp Image 2025-12-19 at 9 01 30 PM](https://github.com/user-attachments/assets/5ee838b1-eadb-4c91-ad59-89e743df16ca)


* This screenshot displays an example of a `trivy-report.json` in JSON format detailing security findings from container image scanning.
<img width="975" height="526" alt="image" src="https://github.com/user-attachments/assets/f4ad839f-a76d-4aba-ad41-90f4fdad66d5" />


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
