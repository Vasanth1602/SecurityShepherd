# Security Shepherd — Security CI/CD Integration

This document describes the complete DevSecOps / Security CI/CD integration implemented for the OWASP Security Shepherd project using Jenkins, SonarQube, and OWASP Dependency-Check.

The purpose of this setup is to automatically perform:

- Application build
- Unit testing
- Software Composition Analysis (SCA)
- Static Application Security Testing (SAST)
- SonarQube Quality Gate validation
- Security report generation
- Jenkins report archiving

The guide starts from forking the Security Shepherd repository and continues through the complete Jenkins pipeline and successful build.

---

# 1. Overview

## 1.1 Project

This implementation is based on:

**OWASP Security Shepherd**

Security Shepherd is an intentionally vulnerable web application designed for security training and testing.

Because the application intentionally contains vulnerabilities, security scanners are expected to report a significant number of findings.

The purpose of this CI/CD integration is not to make Security Shepherd vulnerability-free. Instead, the purpose is to demonstrate how security testing can be integrated into an automated CI/CD pipeline.

---

# 2. Security CI/CD Architecture

The implemented pipeline follows this general flow:

```text
                         GitHub
                            |
                            | Push / Commit
                            v
                    +---------------+
                    |    Jenkins    |
                    +-------+-------+
                            |
                            v
                       Checkout
                            |
                            v
                      Maven Build
                            |
                            v
                       Unit Tests
                            |
                            v
                 OWASP Dependency-Check
                            |
                            | SCA
                            v
                    SCA XML / HTML
                         Reports
                            |
                            v
                       SonarQube
                            |
                            | SAST
                            v
                     Quality Gate
                            |
                            v
                    Jenkins Build Result
                            |
                            v
                    Archive Reports
```

---

# 3. Security Tools Used

The implementation uses the following major components.

| Component | Purpose |
|---|---|
| GitHub | Source code repository |
| Jenkins | CI/CD pipeline orchestration |
| Maven | Build and test automation |
| Java 17 | Java runtime / build environment |
| SonarQube | Static Application Security Testing (SAST) and code-quality analysis |
| OWASP Dependency-Check | Software Composition Analysis (SCA) |
| NVD | Vulnerability data used by Dependency-Check |
| JUnit | Unit test result reporting in Jenkins |
| WSL2 | Linux environment used for local Security Shepherd build |
| Docker Desktop | Container/Docker support for Security Shepherd local development |

---

# 4. SAST and SCA

This project uses two different security analysis approaches.

## 4.1 SAST — SonarQube

SonarQube analyzes the application's source code.

It can identify issues related to:

- Security
- Reliability
- Maintainability
- Code quality
- Security hotspots
- Duplicated code
- Test coverage

SonarQube does not primarily inspect third-party dependency versions. It analyzes the application source code.

---

## 4.2 SCA — OWASP Dependency-Check

OWASP Dependency-Check analyzes third-party dependencies used by the application.

It attempts to identify known vulnerabilities in dependencies by comparing dependency information against vulnerability databases such as the National Vulnerability Database (NVD).

SCA is therefore different from SAST:

```text
SAST
    |
    +--> Analyze application source code
    |
    +--> SonarQube


SCA
    |
    +--> Analyze third-party dependencies
    |
    +--> OWASP Dependency-Check
```

Using both provides broader security coverage.

---

# 5. Repository Setup

## 5.1 Fork Security Shepherd

Start by opening the original OWASP Security Shepherd repository on GitHub.

Fork the repository into your own GitHub account.

The fork used for this implementation is:

```text
https://github.com/Vasanth1602/SecurityShepherd
```

The working branch is:

```text
dev
```

After forking, the repository contains the original Security Shepherd project together with the DevSecOps integration.

---

# 6. Clone the Fork

Clone your fork locally.

Example:

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/SecurityShepherd.git
```

Enter the project directory:

```bash
cd SecurityShepherd
```

Checkout the development branch:

```bash
git checkout dev
```

Verify the current branch:

```bash
git branch
```

The expected working branch is:

```text
dev
```

---

# 7. Git Remote Configuration

The local repository can use two remotes:

```text
origin
    |
    +--> Your GitHub fork

upstream
    |
    +--> Original OWASP Security Shepherd repository
```

Example:

```bash
git remote -v
```

The `origin` remote should point to your fork.

The `upstream` remote can point to the original OWASP repository.

This allows changes from the original project to be retrieved when required.

---

# 8. Local Development Environment

The local environment used during implementation included:

- Windows
- WSL2
- Ubuntu 24.04
- Java 17
- Maven 3.9.11
- Docker Desktop

---

# 9. Java 17

Security Shepherd was built using Java 17.

Verify Java:

```bash
java -version
```

The environment should report Java 17.

Example:

```text
openjdk version "17..."
```

Java 17 is also configured as a Jenkins global tool.

The Jenkins tool name used in this implementation is:

```text
JDK-17
```

---

# 10. Maven

Maven is used to compile, package, and test the application.

Verify Maven locally:

```bash
mvn -version
```

The local Maven version used during the implementation was:

```text
Apache Maven 3.9.11
```

Jenkins uses the configured Maven installation named:

```text
Maven-3.9
```

---

# 11. WSL2

The Security Shepherd project contains Linux shell scripts used by parts of its Docker/Maven workflow.

One of the scripts involved is:

```text
docker/scripts/convert-sql-scripts.sh
```

This caused a problem when the Docker Maven profile was executed directly from Windows because the build process expects a Linux shell environment.

The project was therefore successfully built using WSL2 Ubuntu.

---

# 12. Docker Desktop

Docker Desktop was installed and WSL2 integration was enabled.

Docker is useful for the Security Shepherd application's containerized/local environment.

However, Docker is not required for the Jenkins SAST/SCA pipeline described in this document.

The Jenkins pipeline intentionally uses the normal Maven build rather than the Docker packaging profile.

---

# 13. Why the Jenkins Pipeline Does Not Use the Docker Maven Profile

The Security Shepherd Docker Maven profile invokes Linux-specific shell scripts.

For example:

```text
docker/scripts/convert-sql-scripts.sh
```

Using that profile from a Windows Jenkins environment can introduce unnecessary platform-specific problems.

For SAST and SCA, Docker packaging is not required.

Therefore, the Jenkins pipeline uses:

```bash
mvn clean package -B -DskipTests
```

instead of the Docker Maven profile.

This keeps the security analysis pipeline focused on:

```text
Source Code
    |
    +--> Compile / Package
    |
    +--> Unit Tests
    |
    +--> Dependency-Check
    |
    +--> SonarQube
```

Docker can still be used separately when the complete Security Shepherd application environment needs to be executed.

---

# 14. Verify the Local Build

Before configuring Jenkins, verify that the application can be built.

Run:

```bash
mvn clean package -B -DskipTests
```

A successful build produces the application WAR under:

```text
target/
```

including:

```text
target/owaspSecurityShepherd.war
```

---

# 15. Run Unit Tests Locally

Run:

```bash
mvn test -B
```

The successful test execution during this implementation produced:

```text
Tests run: 169
Failures: 0
Errors: 0
Skipped: 2
```

This confirms that the test suite can execute successfully before introducing Jenkins.

---

# 16. SonarQube Setup

SonarQube is used as the SAST and code-quality analysis platform.

The local SonarQube server used during this implementation was:

```text
http://localhost:9000
```

---

# 17. Start SonarQube

Start the SonarQube server according to the installation method being used.

After SonarQube starts, open:

```text
http://localhost:9000
```

Confirm that the SonarQube web interface is accessible.

---

# 18. SonarQube Initial Login

Use the SonarQube administrator account to access the SonarQube dashboard.

If this is a new installation, complete the initial setup before creating the project.

---

# 19. Create the SonarQube Project

Create a new project in SonarQube.

Use:

```text
Project Name:
Security Shepherd
```

Use:

```text
Project Key:
Security-Shepherd
```

The Jenkins pipeline uses the same project key.

This is important because Jenkins and SonarQube must refer to the same project.

---

# 20. SonarQube Project Configuration

The pipeline passes the following project information to SonarQube:

```text
sonar.projectKey=Security-Shepherd
sonar.projectName=Security Shepherd
```

The Java source version is configured as:

```text
sonar.java.source=17
```

The source directory is:

```text
src/main/java
```

The test directory is:

```text
src/test/java
```

The pipeline excludes:

```text
mobile/**
src/main/resources/database/**
```

from SonarQube source analysis.

---

# 21. Generate a SonarQube Authentication Token

A token is required so Jenkins can authenticate with SonarQube.

In SonarQube:

```text
Account / My Account
        |
        v
Security
        |
        v
Generate Token
```

Generate a token for Jenkins.

Do not commit the token into Git.

Do not place the actual token directly inside the Jenkinsfile.

The token is stored securely in Jenkins credentials.

---

# 22. Jenkins SonarQube Credential

In Jenkins, create a credential for the SonarQube token.

Navigate to:

```text
Jenkins
  |
  +--> Manage Jenkins
        |
        +--> Credentials
```

Create a credential with:

```text
Kind:
Secret text
```

Use:

```text
ID:
sonarqube-token
```

The actual token value must be entered into the secret field.

The value itself must never be committed to GitHub.

The Jenkinsfile refers to the credential indirectly through the SonarQube server configuration.

---

# 23. Configure SonarQube Server in Jenkins

Navigate to:

```text
Manage Jenkins
    |
    +--> System
```

Find:

```text
SonarQube servers
```

Add a SonarQube server.

Configure:

```text
Name:
SonarQube
```

Use:

```text
Server URL:
http://localhost:9000
```

Select the previously created Jenkins credential:

```text
sonarqube-token
```

The resulting configuration is conceptually:

```text
Jenkins
   |
   +--> SonarQube server
            |
            +--> Name: SonarQube
            |
            +--> URL: http://localhost:9000
            |
            +--> Credential: sonarqube-token
```

The pipeline later uses:

```groovy
withSonarQubeEnv('SonarQube')
```

to load this configuration.

---

# 24. SonarQube Webhook

The Quality Gate stage requires Jenkins to receive the analysis result from SonarQube.

SonarQube therefore needs a webhook pointing to Jenkins.

In SonarQube, navigate to:

```text
Administration
    |
    +--> Configuration
          |
          +--> Webhooks
```

Create a webhook.

Use:

```text
http://localhost:8080/sonarqube-webhook/
```

The important endpoint is:

```text
/sonarqube-webhook/
```

The trailing slash should be retained.

---

# 25. Localhost Webhook Validation

Because this implementation uses a local Jenkins/SonarQube environment, the SonarQube webhook configuration initially rejected the localhost target because of URL validation/security restrictions.

For this local lab setup, localhost webhook validation was disabled so that the following could be used:

```text
http://localhost:8080/sonarqube-webhook/
```

This is a **local/lab-specific configuration**.

For a production deployment, Jenkins should normally be exposed through an appropriate reachable hostname or internal network address instead of relying on localhost.

For example:

```text
https://jenkins.example.com/sonarqube-webhook/
```

The exact production URL depends on the organization's Jenkins deployment.

---

# 26. Why the SonarQube Webhook Is Required

The pipeline contains:

```groovy
waitForQualityGate abortPipeline: true
```

Jenkins needs to know when SonarQube has completed the analysis and what Quality Gate result was returned.

The communication flow is:

```text
Jenkins
   |
   | Start SonarQube analysis
   v
SonarQube
   |
   | Analyze project
   |
   | Calculate Quality Gate
   v
SonarQube Webhook
   |
   | POST result
   v
Jenkins
   |
   v
waitForQualityGate
```

Without the webhook, Jenkins may wait indefinitely or fail to receive the Quality Gate result correctly.

---

# 27. Jenkins Installation

Install Jenkins on the machine that will execute the pipeline.

The Jenkins installation used during this implementation runs locally.

The Jenkins web interface was accessed through:

```text
http://localhost:8080
```

---

# 28. Jenkins Plugins

The Jenkins installation requires the plugins needed by the pipeline.

The important plugins/components used for this implementation include:

- Pipeline
- Git integration
- Maven integration
- SonarQube Scanner for Jenkins
- OWASP Dependency-Check
- JUnit / test result reporting

The exact plugin versions may change as Jenkins is updated.

---

# 29. SonarQube Jenkins Integration

Install/configure the Jenkins SonarQube integration plugin.

This provides pipeline functionality such as:

```groovy
withSonarQubeEnv(...)
```

and:

```groovy
waitForQualityGate(...)
```

These are required by the Jenkinsfile.

---

# 30. OWASP Dependency-Check Jenkins Plugin

Install the:

```text
OWASP Dependency-Check
```

Jenkins plugin.

The Jenkins plugin version used during this implementation was:

```text
5.6.3
```

This plugin provides Jenkins integration for running Dependency-Check and publishing its reports.

---

# 31. Dependency-Check Plugin vs Dependency-Check Scanner

These are two different components.

### Jenkins Dependency-Check Plugin

The Jenkins plugin integrates Dependency-Check with Jenkins.

It provides functionality for:

- invoking Dependency-Check
- handling scanner installation
- publishing Dependency-Check reports

### Dependency-Check Scanner

The scanner is the actual OWASP Dependency-Check engine that analyzes dependencies.

The Jenkins installation was configured with:

```text
Installation Name:
OWASP-DC
```

The intended configured scanner version was:

```text
13.0.0
```

However, the Jenkins-generated report examined during the successful run reports:

```text
Dependency-Check Version:
12.2.2
```

Therefore, the exact scanner version actually used by that successful report should be verified in Jenkins before treating `13.0.0` as the definitive runtime version.

This distinction should be retained when troubleshooting or upgrading the environment.

---

# 32. Jenkins Global Tools

Navigate to:

```text
Manage Jenkins
    |
    +--> Tools
```

Configure the following tools.

---

## 32.1 JDK

Add/configure:

```text
Name:
JDK-17
```

The pipeline references this exact name:

```groovy
tools {
    jdk 'JDK-17'
}
```

The name must match exactly.

---

## 32.2 Maven

Configure:

```text
Name:
Maven-3.9
```

The pipeline references:

```groovy
maven 'Maven-3.9'
```

Again, the name must match the Jenkinsfile.

---

## 32.3 OWASP Dependency-Check

Configure a Dependency-Check installation:

```text
Name:
OWASP-DC
```

Enable automatic installation.

The configured source was GitHub.

The scanner version selected during setup was intended to be:

```text
13.0.0
```

The exact version should be verified under the Jenkins tool configuration if the environment has since changed.

---

# 33. Jenkins Credentials for NVD

Dependency-Check uses NVD vulnerability data.

An NVD API key was configured for the Jenkins Dependency-Check scan.

Create a Jenkins credential.

Navigate to:

```text
Manage Jenkins
    |
    +--> Credentials
```

Create:

```text
Kind:
Secret text
```

Use:

```text
ID:
NVD_API_KEY
```

Enter the actual NVD API key as the secret value.

Never put the API key inside:

- Jenkinsfile
- Git repository
- README
- shell scripts
- public documentation

The Jenkinsfile only references the credential ID:

```text
NVD_API_KEY
```

---

# 34. Why the NVD API Key Is Used

Dependency-Check needs vulnerability data.

The NVD API is used to retrieve/update vulnerability information.

Using an API key helps Dependency-Check interact with the NVD API under the applicable API access limits.

The key is stored in Jenkins credentials rather than in source control.

---

# 35. Dependency-Check Data and Cache

Dependency-Check maintains vulnerability data locally.

The first execution on a completely fresh environment can take significantly longer because the vulnerability database needs to be populated.

A Jenkins installation may maintain data under the Jenkins tool installation directory.

The environment used during troubleshooting showed a Dependency-Check data location similar to:

```text
C:\ProgramData\Jenkins\.jenkins\tools\
org.jenkinsci.plugins.DependencyCheck.tools.DependencyCheckInstallation\
OWASP-DC\data
```

A better long-term approach is to use a dedicated persistent data directory rather than depending on a tool installation directory.

For example:

```text
C:\DependencyCheckData
```

This prevents scanner upgrades or tool reinstallations from unnecessarily replacing the vulnerability database.

---

# 36. Recommended Dependency-Check Data Strategy

For a simple local lab:

```text
Jenkins
   |
   +--> Persistent Dependency-Check data
```

For a larger production environment, a better architecture is:

```text
                 Dependency-Check DB Updater
                            |
                            | Update
                            v
                 Persistent Shared Data
                            |
                 +----------+----------+
                 |                     |
                 v                     v
             Jenkins 1             Jenkins 2
                 |                     |
                 +---- Dependency-Check
```

A dedicated updater can periodically refresh the vulnerability data while CI pipelines use the existing database.

This avoids every build independently downloading the entire vulnerability database.

---

# 37. Important Dependency-Check Version Consideration

The Dependency-Check database/data directory should be compatible with the scanner version being used.

If the local Dependency-Check installation and Jenkins Dependency-Check installation use different scanner versions, do not blindly copy the database from one environment to the other.

First verify:

```text
Dependency-Check version
```

in both environments.

The successful Jenkins-generated report used during this implementation states:

```text
Dependency-Check Version: 12.2.2
```

Therefore, verify the actual Jenkins scanner version before seeding Jenkins with an existing local database.

---

# 38. Configure the Jenkins Pipeline Job

Create a new Jenkins job.

Example job name:

```text
SecurityShepherd-SonarQube
```

Select:

```text
Pipeline
```

---

# 39. Configure Pipeline from SCM

The pipeline is stored in the Git repository as:

```text
Jenkinsfile
```

In Jenkins:

```text
Pipeline
    |
    +--> Definition:
         Pipeline script from SCM
```

Select:

```text
SCM:
Git
```

Repository URL:

```text
https://github.com/Vasanth1602/SecurityShepherd.git
```

For another user/fork, replace this with:

```text
https://github.com/<YOUR_GITHUB_USERNAME>/SecurityShepherd.git
```

---

# 40. Configure Git Branch

The configured branch is:

```text
*/dev
```

This means Jenkins builds the `dev` branch.

If another branch is required, change the branch specification accordingly.

---

# 41. Configure Jenkinsfile Path

Set:

```text
Script Path:
Jenkinsfile
```

The Jenkinsfile must be present in the repository root.

The final repository structure should therefore contain:

```text
SecurityShepherd/
│
├── Jenkinsfile
├── SECURITY_CI_CD.md
├── README.md
├── pom.xml
├── src/
└── ...
```

---

# 42. Pipeline Stages

The pipeline contains the following logical stages:

```text
1. Checkout
2. Build
3. Unit Tests
4. OWASP Dependency-Check
5. SonarQube Analysis
6. Quality Gate
7. Post / Archive
```

Dependency-Check report publishing is performed as part of the Dependency-Check stage.

---

# 43. Complete Jenkinsfile

The Jenkinsfile used for the implementation is:

```groovy
/**
 * Jenkins Declarative Pipeline — OWASP Security Shepherd
 * DevSecOps / SCA + SAST pipeline:
 * compile → unit test → Dependency-Check SCA → SonarQube SAST
 * → Quality Gate → archive
 */

pipeline {

    agent any

    tools {
        jdk   'JDK-17'
        maven 'Maven-3.9'
    }

    environment {
        SONAR_PROJECT_KEY  = 'Security-Shepherd'
        SONAR_PROJECT_NAME = 'Security Shepherd'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Branch : ${env.GIT_BRANCH}"
                echo "Commit : ${env.GIT_COMMIT}"
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -B -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                bat 'mvn test -B'
            }

            post {
                always {
                    junit testResults: 'target/surefire-reports/**/*.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('OWASP Dependency-Check') {
            steps {

                dependencyCheck additionalArguments:
                    '--project "Security Shepherd" --format XML --format HTML',
                    nvdCredentialsId: 'NVD_API_KEY',
                    odcInstallation: 'OWASP-DC'

                dependencyCheckPublisher pattern:
                    'dependency-check-report.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('SonarQube') {

                    bat """
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar -B ^
                            -Dsonar.projectKey=%SONAR_PROJECT_KEY% ^
                            -Dsonar.projectName="%SONAR_PROJECT_NAME%" ^
                            -Dsonar.java.source=17 ^
                            -Dsonar.sources=src/main/java ^
                            -Dsonar.tests=src/test/java ^
                            -Dsonar.exclusions=mobile/**,src/main/resources/database/**
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {

                timeout(time: 10, unit: 'MINUTES') {

                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {

        always {

            archiveArtifacts(
                artifacts: 'target/surefire-reports/**, dependency-check-report.xml, dependency-check-report.html, .scannerwork/report-task.txt, target/*.war',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }

        success {
            echo 'Pipeline succeeded.'
        }

        unstable {
            echo 'Pipeline unstable — check test results or Quality Gate.'
        }

        failure {
            echo 'Pipeline failed — review stage logs above.'
        }
    }
}
```

---

# 44. Explanation of Jenkinsfile

## 44.1 Pipeline

```groovy
pipeline {
```

Defines a Jenkins Declarative Pipeline.

---

# 45. Jenkins Agent

```groovy
agent any
```

Allows Jenkins to execute the pipeline on any available Jenkins agent that satisfies the required configuration.

---

# 46. Jenkins Tools

```groovy
tools {
    jdk   'JDK-17'
    maven 'Maven-3.9'
}
```

These names refer to the tools configured under:

```text
Manage Jenkins → Tools
```

They must match exactly.

---

# 47. Environment Variables

```groovy
environment {
    SONAR_PROJECT_KEY  = 'Security-Shepherd'
    SONAR_PROJECT_NAME = 'Security Shepherd'
}
```

These values are passed to the SonarQube scanner.

---

# 48. Pipeline Options

The pipeline uses:

```groovy
buildDiscarder(logRotator(
    numToKeepStr: '20',
    artifactNumToKeepStr: '10'
))
```

This prevents Jenkins from retaining unlimited old builds and artifacts.

---

The pipeline has a maximum execution time of:

```text
60 minutes
```

using:

```groovy
timeout(time: 60, unit: 'MINUTES')
```

---

Timestamps are enabled:

```groovy
timestamps()
```

This makes Jenkins console logs easier to understand.

---

Concurrent builds are disabled:

```groovy
disableConcurrentBuilds()
```

This prevents multiple executions of the same pipeline from running concurrently.

---

# 49. Checkout Stage

```groovy
stage('Checkout')
```

The Jenkins job is configured with Git SCM.

Jenkins performs the actual source checkout as part of the SCM pipeline configuration.

The stage prints:

```text
Branch
Commit
```

for traceability.

---

# 50. Build Stage

The build command is:

```bash
mvn clean package -B -DskipTests
```

Meaning:

```text
clean
    |
    +--> Remove previous build output

package
    |
    +--> Compile and package application

-B
    |
    +--> Batch mode

-DskipTests
    |
    +--> Do not execute unit tests during packaging
```

Tests are intentionally executed in a separate stage.

---

# 51. Why Tests Are Separate

The pipeline first builds the application:

```text
Build
```

and then runs:

```text
Unit Tests
```

This provides clearer pipeline visibility.

Instead of combining everything into:

```text
mvn package
```

the pipeline makes the stages explicit:

```text
Build
   |
   v
Unit Tests
```

---

# 52. Unit Test Stage

The command is:

```bash
mvn test -B
```

Jenkins then collects the Surefire XML results:

```text
target/surefire-reports/**/*.xml
```

The results are published using:

```groovy
junit
```

This allows Jenkins to display:

- Tests run
- Passed tests
- Failed tests
- Errors
- Skipped tests
- Test history

---

# 53. Unit Test Result

The successful test execution achieved:

```text
Tests run: 169
Failures: 0
Errors: 0
Skipped: 2
```

Therefore:

```text
Unit Tests: PASS
```

---

# 54. OWASP Dependency-Check Stage

The Dependency-Check invocation is:

```groovy
dependencyCheck additionalArguments:
    '--project "Security Shepherd" --format XML --format HTML',
    nvdCredentialsId: 'NVD_API_KEY',
    odcInstallation: 'OWASP-DC'
```

---

# 55. Dependency-Check Project Name

The project name passed to Dependency-Check is:

```text
Security Shepherd
```

This identifies the project in the generated report.

---

# 56. Dependency-Check Output Formats

The command specifies:

```text
--format XML
--format HTML
```

Therefore, Dependency-Check generates:

```text
dependency-check-report.xml
dependency-check-report.html
```

The XML report is used by Jenkins for publishing.

The HTML report can be opened to inspect the detailed findings.

---

# 57. Dependency-Check NVD Credential

The Jenkinsfile contains:

```groovy
nvdCredentialsId: 'NVD_API_KEY'
```

This does not contain the actual API key.

It references the Jenkins credential:

```text
NVD_API_KEY
```

This keeps the secret outside source control.

---

# 58. Dependency-Check Scanner Installation

The Jenkinsfile contains:

```groovy
odcInstallation: 'OWASP-DC'
```

This refers to the Dependency-Check scanner configured under Jenkins:

```text
Manage Jenkins
    |
    +--> Tools
          |
          +--> Dependency-Check
```

with installation name:

```text
OWASP-DC
```

---

# 59. Dependency-Check Publisher

After scanning:

```groovy
dependencyCheckPublisher pattern:
    'dependency-check-report.xml'
```

Jenkins reads the generated XML report and publishes the Dependency-Check results.

This allows the findings to be associated with the Jenkins build.

---

# 60. Dependency-Check Thresholds

The current implementation intentionally does not use:

```text
--failOnCVSS
```

and does not configure Dependency-Check thresholds to automatically fail the build based on vulnerability severity.

This was intentional during the initial integration.

The first objective was:

```text
Scan
   |
   v
Generate Report
   |
   v
Publish Findings
   |
   v
Establish Baseline
```

rather than immediately making the build fail because Security Shepherd is intentionally vulnerable.

---

# 61. Why the Pipeline Does Not Fail on Every Vulnerability

Security Shepherd is intentionally vulnerable.

Therefore, configuring an immediate rule such as:

```text
fail on CVSS >= X
```

could cause the pipeline to fail continuously.

The current setup focuses first on visibility and reporting.

A future production pipeline can introduce thresholds based on organizational requirements.

---

# 62. SonarQube Analysis Stage

The SonarQube stage uses:

```groovy
withSonarQubeEnv('SonarQube')
```

This loads the Jenkins SonarQube server configuration named:

```text
SonarQube
```

The scanner then executes.

---

# 63. SonarQube Maven Scanner

The pipeline uses the explicit Maven plugin coordinates:

```bash
mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar
```

instead of:

```bash
mvn sonar:sonar
```

This was necessary because the shorter:

```bash
mvn sonar:sonar
```

initially failed with:

```text
No plugin found for prefix 'sonar'
```

Using the explicit plugin coordinates resolved the issue.

---

# 64. SonarQube Parameters

The pipeline sends:

```text
-Dsonar.projectKey=Security-Shepherd
```

Project key:

```text
Security-Shepherd
```

Project name:

```text
Security Shepherd
```

Java version:

```text
17
```

Source directory:

```text
src/main/java
```

Test directory:

```text
src/test/java
```

Exclusions:

```text
mobile/**
src/main/resources/database/**
```

---

# 65. SonarQube Quality Gate

After SonarQube analysis, Jenkins executes:

```groovy
waitForQualityGate abortPipeline: true
```

This tells Jenkins to wait for the SonarQube Quality Gate result.

If the Quality Gate fails, Jenkins aborts the pipeline.

The timeout is:

```text
10 minutes
```

---

# 66. SonarQube Quality Gate vs Vulnerability Count

A very important point:

```text
Quality Gate Passed
```

does **not** necessarily mean:

```text
No vulnerabilities
```

SonarQube Quality Gates evaluate configured conditions.

A project can contain many existing vulnerabilities/issues while still passing the configured Quality Gate.

Security Shepherd is intentionally vulnerable, so a large number of findings is expected.

---

# 67. SonarQube Results

The Security Shepherd analysis produced approximately:

```text
Security Issues:       235
Reliability Issues:    109
Maintainability:       ~3,300
Coverage:              0%
Duplications:          25.3%
Security Rating:       E
```

The SonarQube Quality Gate nevertheless returned:

```text
PASSED
```

This demonstrates why the Quality Gate status must be interpreted separately from the total number of findings.

---

# 68. Post-Build Actions

The pipeline archives reports using:

```groovy
archiveArtifacts
```

The configured artifacts include:

```text
target/surefire-reports/**
dependency-check-report.xml
dependency-check-report.html
.scannerwork/report-task.txt
target/*.war
```

---

# 69. Archived Test Reports

Jenkins archives:

```text
target/surefire-reports/**
```

These files contain the JUnit/Surefire test results.

---

# 70. Archived Dependency-Check Reports

Jenkins archives:

```text
dependency-check-report.xml
dependency-check-report.html
```

The HTML file provides a human-readable security report.

The XML file is useful for Jenkins report processing.

---

# 71. Archived WAR

The pipeline also archives:

```text
target/*.war
```

This allows the generated application WAR to be retained with the Jenkins build.

---

# 72. Scanner Report Task File

The current archive configuration also references:

```text
.scannerwork/report-task.txt
```

The archive uses:

```groovy
allowEmptyArchive: true
```

Therefore, if this file is not produced by the current scanner configuration, it does not cause the build to fail.

The Maven Sonar scanner may instead use its Maven-specific working/output directories.

This archive entry is therefore non-critical.

---

# 73. Build Retention

The pipeline retains:

```text
20 builds
```

and:

```text
10 sets of archived artifacts
```

This is controlled by:

```groovy
buildDiscarder(logRotator(
    numToKeepStr: '20',
    artifactNumToKeepStr: '10'
))
```

---

# 74. Final Pipeline Flow

The complete Jenkins execution is:

```text
Checkout
   |
   v
Build
   |
   | mvn clean package -B -DskipTests
   v
Unit Tests
   |
   | mvn test -B
   v
OWASP Dependency-Check
   |
   | SCA
   |
   +--> XML
   |
   +--> HTML
   |
   v
Dependency-Check Publisher
   |
   v
SonarQube Analysis
   |
   | SAST
   v
SonarQube Quality Gate
   |
   +---- PASS ----> Continue
   |
   +---- FAIL ----> Abort
   |
   v
Archive Reports
   |
   v
Pipeline Result
```

---

# 75. Successful Jenkins Build

The complete pipeline was successfully executed.

The successful pipeline included:

```text
Checkout       PASS
Build          PASS
Unit Tests     PASS
SCA            PASS
SCA Publisher  PASS
SonarQube      PASS
Quality Gate   PASS
Post Actions   PASS
```

Final result:

```text
SUCCESS
```

---

# 76. Jenkins SCA Report Results

The Jenkins-generated Dependency-Check HTML report used during validation reported:

```text
Project:
Security Shepherd
```

```text
Dependency-Check Version:
12.2.2
```

```text
Dependencies Scanned:
158
```

```text
Unique Dependencies:
135
```

```text
Vulnerable Dependencies:
25
```

```text
Vulnerabilities Found:
62
```

```text
Vulnerabilities Suppressed:
0
```

The report was successfully generated and published by Jenkins.

---

# 77. Dependency-Check NVD Status

The report contained NVD information indicating that the NVD data had been checked on:

```text
2026-09-10T12:12:11+0530
```

and the NVD data's last modified timestamp was reported as:

```text
2026-09-10T06:17:06Z
```

The exact values will naturally change as the vulnerability database is updated.

---

# 78. Dependency-Check Network Warning

During the first Jenkins SCA execution, Dependency-Check attempted to update vulnerability information from external sources.

The environment experienced network-related failures including NVD API retry failures and failures involving external services such as:

```text
raw.githubusercontent.com
dependency-check.github.io
www.cisa.gov
search.maven.org
```

The NVD update encountered:

```text
NvdApiRetryExceededException
```

Dependency-Check subsequently continued using locally available/cached data and generated the report.

---

# 79. Important Interpretation of the SCA Warning

The successful generation of a report does not mean every external analyzer was fully available.

The Jenkins-generated report specifically contained an analysis exception indicating:

```text
Could not connect to Central search.
Analysis failed; disabling Central analyzer.
```

The report warns that this can potentially result in:

```text
false positives
```

or:

```text
false negatives
```

for analysis performed by that analyzer.

Therefore, the correct interpretation is:

```text
Dependency-Check scan completed successfully
        +
Report generated successfully
        +
Jenkins published the report
        +
NVD-backed findings are available
        -
Some external analyzers may have been unavailable
```

This should be considered when interpreting the results.

---

# 80. Dependency-Check Local vs Jenkins Execution

A locally configured Dependency-Check environment may produce results faster if its vulnerability database is already fully populated.

A fresh Jenkins environment may initially be slower because it must download/update vulnerability data.

Therefore:

```text
Local Dependency-Check
    |
    +--> Existing database/cache
    |
    +--> Faster


Fresh Jenkins Dependency-Check
    |
    +--> Download/update data
    |
    +--> Slower
```

This is expected.

---

# 81. Recommended Improvement for Dependency-Check

For a more stable CI environment, maintain a persistent Dependency-Check data directory.

For example:

```text
C:\DependencyCheckData
```

or another persistent location appropriate for the Jenkins environment.

The important point is that the vulnerability database should survive:

- Jenkins builds
- scanner restarts
- Jenkins restarts
- scanner/tool upgrades where compatible

The exact implementation depends on the Jenkins deployment architecture.

---

# 82. Recommended Production SCA Architecture

For a larger environment, use:

```text
                 Scheduled Update Job
                         |
                         v
              Dependency-Check Data
                  Persistent Storage
                         |
             +-----------+-----------+
             |                       |
             v                       v
          Jenkins                  Jenkins
          Pipeline                 Pipeline
             |                       |
             +------ SCA ------------+
```

The update process can refresh the vulnerability database periodically, while normal application builds reuse the available data.

This reduces unnecessary repeated downloads.

---

# 83. Why Suppressions Were Not Added

The current implementation reports:

```text
Vulnerabilities Suppressed:
0
```

No suppressions were added merely to make the build appear clean.

This is intentional.

A vulnerability should only be suppressed when there is a documented and justified reason, such as:

- confirmed false positive
- accepted risk
- dependency analysis limitation
- documented compensating control

Suppressions should not be used simply to make a security report look better.

---

# 84. Why Severity Thresholds Were Not Added Initially

No:

```text
--failOnCVSS
```

threshold was configured during the initial baseline implementation.

This was done because Security Shepherd intentionally contains vulnerabilities.

The initial goal was:

```text
Visibility
    +
Reporting
    +
Baseline
```

After the organization understands the findings, severity thresholds can be introduced.

For example:

```text
Fail build for new Critical vulnerabilities
```

or:

```text
Fail build for new High/Critical vulnerabilities
```

The exact policy should be determined by the project's security requirements.

---

# 85. Recommended Future Security Gate

A mature pipeline could evolve into:

```text
Build
   |
Unit Tests
   |
SCA
   |
SAST
   |
   +--> Critical/High policy check
   |
   +--> Quality Gate
   |
   v
Deployment
```

Instead of blocking on every historical vulnerability, a production team may choose to enforce policies primarily on newly introduced vulnerabilities.

---

# 86. Troubleshooting — SonarQube Not Running

If Jenkins reports that it cannot connect to SonarQube, verify that SonarQube is running.

Open:

```text
http://localhost:9000
```

If the page is unavailable, start SonarQube before running the Jenkins pipeline.

The Jenkins configuration expects:

```text
http://localhost:9000
```

---

# 87. Troubleshooting — SonarQube URL

Check:

```text
Manage Jenkins
    |
    +--> System
          |
          +--> SonarQube servers
```

Verify:

```text
Name:
SonarQube
```

and:

```text
Server URL:
http://localhost:9000
```

The Jenkinsfile must use the same server name:

```groovy
withSonarQubeEnv('SonarQube')
```

---

# 88. Troubleshooting — SonarQube Token

If authentication fails:

1. Verify the SonarQube token exists.
2. Verify the Jenkins credential exists.
3. Verify the credential ID is:

```text
sonarqube-token
```

4. Verify the token has not expired/revoked.
5. Verify the Jenkins SonarQube server configuration uses the credential.

Never put the token directly into the Jenkinsfile.

---

# 89. Troubleshooting — SonarQube Prefix Error

An earlier command:

```bash
mvn sonar:sonar
```

failed with:

```text
No plugin found for prefix 'sonar'
```

The working command is:

```bash
mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar
```

Therefore the Jenkinsfile uses the explicit plugin coordinates.

---

# 90. Troubleshooting — Quality Gate Waiting

If Jenkins stays at:

```text
Quality Gate
```

check the SonarQube webhook.

SonarQube should contain:

```text
http://localhost:8080/sonarqube-webhook/
```

Also verify that Jenkins is accessible from SonarQube.

In a production environment, `localhost` should generally not be used unless Jenkins and SonarQube are running in the same network context where localhost resolves correctly.

---

# 91. Troubleshooting — Dependency-Check NVD Failure

If the Jenkins log shows:

```text
NvdApiRetryExceededException
```

check:

- Internet connectivity
- NVD API availability
- NVD API key
- Jenkins credential
- Dependency-Check version
- NVD API rate limits
- Dependency-Check data/cache

The API credential should be:

```text
NVD_API_KEY
```

---

# 92. Troubleshooting — Dependency-Check External Sources

Dependency-Check can use multiple external data sources/analyzers.

Failures may involve services such as:

```text
NVD
Maven Central
RetireJS
CISA KEV
Hosted suppressions
```

A failure in one analyzer does not necessarily mean that the entire Dependency-Check execution failed.

However, the resulting report should be reviewed for warnings and analysis exceptions.

---

# 93. Troubleshooting — Dependency-Check Report Missing

If Jenkins cannot find:

```text
dependency-check-report.xml
```

verify:

1. Dependency-Check actually executed.
2. The scanner generated the report.
3. The working directory is the Jenkins workspace.
4. XML format was requested.
5. The publisher pattern matches:

```text
dependency-check-report.xml
```

The configured scanner arguments include:

```text
--format XML
--format HTML
```

---

# 94. Troubleshooting — Dependency-Check Installation

If Jenkins cannot find the scanner:

Go to:

```text
Manage Jenkins
    |
    +--> Tools
          |
          +--> Dependency-Check
```

Verify the installation name:

```text
OWASP-DC
```

The Jenkinsfile must use:

```groovy
odcInstallation: 'OWASP-DC'
```

The name is case-sensitive.

---

# 95. Troubleshooting — NVD Credential

If Dependency-Check cannot access NVD correctly:

Check:

```text
Manage Jenkins
    |
    +--> Credentials
```

Verify:

```text
Credential ID:
NVD_API_KEY
```

The Jenkinsfile references:

```groovy
nvdCredentialsId: 'NVD_API_KEY'
```

Do not replace the credential ID with the actual secret.

---

# 96. Troubleshooting — Maven Not Found

If Jenkins reports:

```text
mvn is not recognized
```

or a similar Maven error:

Check:

```text
Manage Jenkins
    |
    +--> Tools
```

Verify:

```text
Maven-3.9
```

exists.

The Jenkinsfile contains:

```groovy
maven 'Maven-3.9'
```

The tool name must match.

---

# 97. Troubleshooting — Java Not Found

If Jenkins reports Java-related errors:

Check:

```text
Manage Jenkins
    |
    +--> Tools
```

Verify:

```text
JDK-17
```

exists.

The Jenkinsfile contains:

```groovy
jdk 'JDK-17'
```

---

# 98. Troubleshooting — Windows Docker Build

If the Docker Maven profile fails on Windows because of:

```text
docker/scripts/convert-sql-scripts.sh
```

use the WSL2 environment for the complete Docker-oriented Security Shepherd build.

For the Jenkins security pipeline, use:

```bash
mvn clean package -B -DskipTests
```

because Docker packaging is not necessary for SAST/SCA.

---

# 99. Troubleshooting — Empty JUnit Results

The pipeline uses:

```groovy
junit testResults:
    'target/surefire-reports/**/*.xml',
    allowEmptyResults: true
```

If Jenkins shows no test results, verify that:

```text
target/surefire-reports/
```

contains XML files.

Run:

```bash
mvn test -B
```

manually and verify the directory.

---

# 100. Jenkins Build Failure Investigation

When a build fails, inspect the Jenkins stage that failed.

Recommended order:

```text
1. Checkout
2. Build
3. Unit Tests
4. Dependency-Check
5. SonarQube
6. Quality Gate
7. Post Actions
```

Do not immediately assume that a security finding caused the build to fail.

The current pipeline does not configure Dependency-Check CVSS failure thresholds.

---

# 101. Understanding the Final Jenkins Result

A successful Jenkins build means that all configured pipeline requirements completed successfully.

It does not mean:

```text
Application is completely secure
```

Instead, it means:

```text
Build completed
+
Tests completed successfully
+
SCA completed
+
SAST completed
+
Configured Quality Gate passed
+
Reports were generated/published
```

Security findings can still exist.

---

# 102. Security Shepherd Is Intentionally Vulnerable

Security Shepherd is specifically designed for security training.

Therefore, the scanners are expected to identify many security issues.

The presence of findings in:

```text
SonarQube
```

or:

```text
Dependency-Check
```

does not indicate that the integration is broken.

In fact, the findings demonstrate that the security tools are analyzing the application.

---

# 103. Example SonarQube Interpretation

The analysis showed approximately:

```text
Security Rating: E
```

along with a large number of security and code-quality findings.

This is expected for an intentionally vulnerable security-training application.

The Quality Gate result:

```text
Passed
```

should therefore be interpreted as:

> The configured SonarQube Quality Gate conditions passed.

It should not be interpreted as:

> Security Shepherd contains no security vulnerabilities.

---

# 104. Example SCA Interpretation

The Dependency-Check report showed:

```text
158 dependencies scanned
135 unique dependencies
25 vulnerable dependencies
62 vulnerabilities
0 suppressed
```

This indicates that Dependency-Check identified vulnerable third-party dependencies.

The findings should be reviewed individually based on:

- CVE
- Severity
- Installed version
- Fixed version
- Dependency path
- Whether the vulnerable dependency is actually used
- Whether a fix/upgrade is available
- Whether the finding is a false positive

---

# 105. Security Report Lifecycle

The intended lifecycle is:

```text
Developer Commit
       |
       v
Jenkins
       |
       v
Build
       |
       v
Test
       |
       v
SCA
       |
       v
SAST
       |
       v
Quality Gate
       |
       v
Reports
       |
       v
Security Review
       |
       v
Remediation
       |
       v
New Commit
       |
       +----------> Jenkins
```

This creates a continuous security feedback loop.

---

# 106. What Was Actually Automated

The Jenkins pipeline automates:

### Build

```text
mvn clean package -B -DskipTests
```

### Tests

```text
mvn test -B
```

### SCA

```text
OWASP Dependency-Check
```

### SAST

```text
SonarQube
```

### Quality Gate

```text
waitForQualityGate
```

### Reporting

```text
JUnit
Dependency-Check Publisher
SonarQube
```

### Artifact retention

```text
Jenkins archiveArtifacts
```

---

# 107. Secrets Management

The pipeline follows the principle of keeping secrets outside source control.

The following are stored as Jenkins credentials:

```text
sonarqube-token
NVD_API_KEY
```

The Jenkinsfile contains only the credential identifiers.

Never commit:

```text
SonarQube token
NVD API key
Passwords
Private keys
Cloud credentials
```

to GitHub.

---

# 108. Current Jenkins Configuration Summary

The main Jenkins configuration is:

| Configuration | Value |
|---|---|
| Jenkins Job | `SecurityShepherd-SonarQube` |
| SCM | Git |
| Repository | `https://github.com/Vasanth1602/SecurityShepherd.git` |
| Branch | `*/dev` |
| Jenkinsfile | `Jenkinsfile` |
| JDK | `JDK-17` |
| Maven | `Maven-3.9` |
| Dependency-Check | `OWASP-DC` |
| SonarQube Server Name | `SonarQube` |
| SonarQube URL | `http://localhost:9000` |
| SonarQube Credential | `sonarqube-token` |
| NVD Credential | `NVD_API_KEY` |
| SonarQube Webhook | `http://localhost:8080/sonarqube-webhook/` |

---

# 109. Current SonarQube Configuration Summary

| Setting | Value |
|---|---|
| Project Name | `Security Shepherd` |
| Project Key | `Security-Shepherd` |
| Server | `http://localhost:9000` |
| Jenkins Credential | `sonarqube-token` |
| Webhook | `http://localhost:8080/sonarqube-webhook/` |
| Java Source | `17` |
| Sources | `src/main/java` |
| Tests | `src/test/java` |
| Exclusions | `mobile/**,src/main/resources/database/**` |

---

# 110. Current Dependency-Check Configuration Summary

| Setting | Value |
|---|---|
| Jenkins Plugin | OWASP Dependency-Check |
| Jenkins Plugin Version Used | `5.6.3` |
| Installation Name | `OWASP-DC` |
| Intended Scanner Version | `13.0.0` |
| Version in Successful Report | `12.2.2` |
| NVD Credential | `NVD_API_KEY` |
| Output | XML + HTML |
| Project | `Security Shepherd` |
| CVSS Failure Threshold | Not configured |
| Suppressions | None |
| Publisher | `dependency-check-report.xml` |

The scanner-version difference should be verified before standardizing the environment.

---

# 111. Current Test Results

The successful unit-test execution produced:

```text
Tests run: 169
Failures: 0
Errors: 0
Skipped: 2
```

Result:

```text
PASS
```

---

# 112. Current SCA Results

The successful Jenkins-generated Dependency-Check report contained:

```text
Dependencies Scanned: 158
Unique Dependencies: 135
Vulnerable Dependencies: 25
Vulnerabilities Found: 62
Vulnerabilities Suppressed: 0
```

Result:

```text
REPORT GENERATED
```

The report also contained a warning related to the unavailable Central Analyzer.

---

# 113. Current SAST Results

The SonarQube analysis produced approximately:

```text
Security Issues:       235
Reliability Issues:    109
Maintainability:       ~3,300
Coverage:              0%
Duplications:          25.3%
Security Rating:       E
```

Quality Gate:

```text
PASSED
```

Again, these findings are expected because Security Shepherd is intentionally vulnerable.

---

# 114. What This Project Demonstrates

This implementation demonstrates a DevSecOps pipeline where security checks are integrated into CI rather than performed only manually.

The pipeline automatically performs:

```text
Source Checkout
       |
       v
Compilation
       |
       v
Unit Testing
       |
       v
Software Composition Analysis
       |
       v
Static Application Security Testing
       |
       v
Quality Gate
       |
       v
Security Reports
```

This allows security analysis to become part of the normal software development workflow.

---

# 115. Why Both SAST and SCA Are Used

Using only one security scanner leaves gaps.

### SonarQube / SAST

Focus:

```text
Application source code
```

Examples:

```text
Insecure coding patterns
Security issues
Security hotspots
Code quality issues
```

### Dependency-Check / SCA

Focus:

```text
Third-party libraries
```

Examples:

```text
Known CVEs
Vulnerable dependency versions
Third-party component risks
```

Therefore:

```text
SAST + SCA
     |
     v
Broader security coverage
```

---

# 116. Why Jenkins Is Used

Jenkins acts as the automation/orchestration layer.

Without Jenkins, a developer might manually run:

```bash
mvn test
```

then:

```text
Dependency-Check
```

then:

```text
SonarQube
```

With Jenkins:

```text
Developer Commit
       |
       v
Jenkins
       |
       +--> Build
       |
       +--> Test
       |
       +--> SCA
       |
       +--> SAST
       |
       +--> Quality Gate
       |
       v
Result
```

The process becomes repeatable.

---

# 117. Reproducing the Setup on Another Machine

To reproduce the setup:

## Step 1

Fork Security Shepherd.

## Step 2

Clone the fork.

## Step 3

Checkout:

```text
dev
```

## Step 4

Install/configure:

```text
Java 17
Maven 3.9.x
Jenkins
SonarQube
```

## Step 5

Configure Jenkins tools:

```text
JDK-17
Maven-3.9
OWASP-DC
```

## Step 6

Install required Jenkins plugins.

## Step 7

Create SonarQube project:

```text
Security Shepherd
```

with:

```text
Security-Shepherd
```

as project key.

## Step 8

Generate SonarQube token.

## Step 9

Store it in Jenkins:

```text
sonarqube-token
```

## Step 10

Configure SonarQube server in Jenkins:

```text
Name:
SonarQube

URL:
http://localhost:9000
```

## Step 11

Configure SonarQube webhook:

```text
http://localhost:8080/sonarqube-webhook/
```

## Step 12

Create NVD API credential:

```text
NVD_API_KEY
```

## Step 13

Configure Dependency-Check:

```text
OWASP-DC
```

## Step 14

Create Jenkins Pipeline job:

```text
SecurityShepherd-SonarQube
```

## Step 15

Configure:

```text
Pipeline script from SCM
```

## Step 16

Configure repository:

```text
https://github.com/<YOUR_USERNAME>/SecurityShepherd.git
```

## Step 17

Configure branch:

```text
*/dev
```

## Step 18

Configure script path:

```text
Jenkinsfile
```

## Step 19

Run the pipeline.

## Step 20

Review:

```text
Jenkins
   |
   +--> Unit Test Results
   |
   +--> Dependency-Check Report
   |
   +--> SonarQube Analysis
   |
   +--> Quality Gate
   |
   +--> Archived Artifacts
```

---

# 118. Production Considerations

The current implementation is primarily a local/lab DevSecOps setup.

Before using it as a production CI/CD security gate, consider the following improvements.

## 118.1 Use a reachable SonarQube/Jenkins URL

Instead of:

```text
http://localhost:8080/sonarqube-webhook/
```

use a properly reachable Jenkins endpoint.

---

## 118.2 Persist Dependency-Check Data

Use persistent storage for the Dependency-Check vulnerability database.

---

## 118.3 Introduce Security Thresholds

After establishing a baseline, introduce policies such as:

```text
Fail on newly introduced Critical vulnerabilities
```

or:

```text
Fail on newly introduced High/Critical vulnerabilities
```

---

## 118.4 Manage False Positives Properly

Use documented Dependency-Check suppressions only when justified.

Do not suppress findings merely to achieve a green build.

---

## 118.5 Secure Jenkins

Use:

- HTTPS
- Authentication
- Authorization
- Least-privilege permissions
- Secure credentials
- Restricted agent access
- Regular plugin updates

---

## 118.6 Secure SonarQube

Use:

- HTTPS
- Authentication
- Appropriate project permissions
- Secure token management
- Controlled webhook access

---

# 119. Future Improvements

Possible improvements include:

```text
1. Persistent Dependency-Check database
2. Scheduled Dependency-Check database updates
3. CVSS-based build thresholds
4. Security policy for Critical/High vulnerabilities
5. Baseline management
6. Improved SonarQube Quality Gate rules
7. Coverage thresholds
8. Automated deployment after security gates
9. Container image scanning
10. Secret scanning
11. Infrastructure-as-Code scanning
12. SBOM generation
13. Security notification integration
14. Pull-request security checks
```

---

# 120. Recommended Mature DevSecOps Pipeline

A future version can evolve into:

```text
                 Developer
                     |
                     v
                  GitHub
                     |
                     v
                  Jenkins
                     |
          +----------+----------+
          |                     |
          v                     v
       Build                 Unit Tests
          |                     |
          +----------+----------+
                     |
                     v
              Dependency-Check
                     |
                     v
                    SCA
                     |
                     v
                 SonarQube
                     |
                     v
                    SAST
                     |
                     v
               Quality Gates
                     |
          +----------+----------+
          |                     |
         FAIL                  PASS
          |                     |
          v                     v
       Stop                  Package
                                |
                                v
                             Deploy
```

---

# 121. Security Principles Followed

This implementation follows several basic DevSecOps principles:

### Security as Code

The security pipeline is defined in:

```text
Jenkinsfile
```

rather than being manually configured only through Jenkins UI.

### Automated Security Testing

SAST and SCA execute automatically as pipeline stages.

### Secrets Outside Source Code

Credentials are stored in Jenkins rather than the Git repository.

### Repeatability

The same pipeline can be executed repeatedly.

### Security Visibility

Reports are published and archived.

### Quality Gate Enforcement

SonarQube can prevent further pipeline progression when the configured Quality Gate fails.

---

# 122. Important Limitations

This setup should not be interpreted as a complete security solution.

SAST and SCA cannot identify every possible security problem.

The pipeline does not replace:

- Manual penetration testing
- Security architecture review
- Threat modeling
- Secure code review
- Runtime testing
- Infrastructure security testing
- Container security testing
- Secret management
- Production monitoring

Instead, it provides automated security checks as part of CI/CD.

---

# 123. Final Result

The completed Security Shepherd DevSecOps pipeline successfully integrates:

```text
GitHub
   +
Jenkins
   +
Maven
   +
JUnit
   +
OWASP Dependency-Check
   +
SonarQube
```

The resulting pipeline automatically performs:

```text
1. Source checkout
2. Maven build
3. Unit tests
4. SCA using OWASP Dependency-Check
5. Dependency-Check report publishing
6. SAST using SonarQube
7. SonarQube Quality Gate validation
8. Test/security report archiving
9. WAR artifact archiving
```

The final validated Jenkins execution completed successfully.

---

# 124. Quick Reference

## Repository

```text
SecurityShepherd
```

## Branch

```text
dev
```

## Jenkins Job

```text
SecurityShepherd-SonarQube
```

## Jenkinsfile

```text
Jenkinsfile
```

## Java

```text
JDK-17
```

## Maven

```text
Maven-3.9
```

## SonarQube

```text
Security Shepherd
```

## SonarQube Project Key

```text
Security-Shepherd
```

## SonarQube Jenkins Server Name

```text
SonarQube
```

## SonarQube URL

```text
http://localhost:9000
```

## SonarQube Jenkins Credential

```text
sonarqube-token
```

## SonarQube Webhook

```text
http://localhost:8080/sonarqube-webhook/
```

## Dependency-Check Installation

```text
OWASP-DC
```

## NVD Credential

```text
NVD_API_KEY
```

## Dependency-Check Reports

```text
dependency-check-report.xml
dependency-check-report.html
```

## Unit Test Reports

```text
target/surefire-reports/
```

## Application Artifact

```text
target/*.war
```

---

# 125. Conclusion

This implementation integrates security testing directly into the Security Shepherd CI pipeline.

The resulting workflow ensures that every pipeline execution can automatically:

```text
Build the application
       ↓
Run unit tests
       ↓
Check third-party dependencies
       ↓
Analyze source code
       ↓
Evaluate the SonarQube Quality Gate
       ↓
Publish security results
       ↓
Archive build and security artifacts
```

The implementation provides a foundation for extending the pipeline into a more mature DevSecOps workflow with vulnerability thresholds, persistent vulnerability databases, automated remediation tracking, additional security scanners, and deployment gates.
