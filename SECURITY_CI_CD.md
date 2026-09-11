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

The Jenkinsfile defines the following explicit pipeline stages:

```text
1. Checkout
2. Build
3. Unit Tests
4. OWASP Dependency-Check
5. SonarQube Analysis
6. Quality Gate
```

After all stages complete, Jenkins executes a `post {}` block (not a stage) that:

- Publishes/archives test reports, Dependency-Check reports, and the WAR artifact
- Logs the overall pipeline result (success / failure / unstable)

Dependency-Check report publishing (`dependencyCheckPublisher`) is executed inside the Dependency-Check stage itself, not in the post block.

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

# 68. OPTIONAL — CNES SonarQube Report Generation

> **This section is OPTIONAL and describes a manual/local workflow only.**
> **CNES Report is NOT part of the current Jenkinsfile or automated pipeline.**

The current Jenkins pipeline automatically produces:

- Dependency-Check XML and HTML reports (archived as Jenkins build artifacts)
- SonarQube SAST results (viewed in the SonarQube dashboard)
- JUnit test results
- Application WAR artifact

SonarQube analysis results are primarily viewed through the SonarQube web dashboard at:

```text
http://localhost:9000
```

CNES Report is a separate command-line utility that can be run **after** a SonarQube analysis has already completed. It connects to an existing SonarQube project and exports the analysis results into standalone editable/exportable report files.

CNES does not perform SAST itself. It is a reporting and export layer on top of existing SonarQube results.

The relationship is:

```text
Jenkins
    |
    +--> SonarQube Analysis
              |
              v
        SonarQube Dashboard       <-- primary results view
              |
              v
        OPTIONAL CNES Report      <-- manual, separate step
              |
              v
     Standalone report files
```

---

# 69. CNES Report Tool

CNES Report is an open-source SonarQube reporting utility available as a standalone JAR.

The JAR used in this implementation is:

```text
sonar-cnes-report-5.0.4.jar
```

The directory containing the JAR (`<CNES_REPORT_DIR>`):

```text
<CNES_REPORT_DIR>\
```

> **Local/lab environment example:**
> `V:\platform-tools\CNES-Report\`

The directory structure at the time of testing:

```text
<CNES_REPORT_DIR>\
|
+-- sonar-cnes-report-5.0.4.jar
+-- node-goat_reports\
+-- reports\
+-- Security-Shepherd-report\
```

Java must be installed and available in the PATH to run the JAR.

---

# 70. CNES Report Command-Line Options

The available options can be verified by running:

```powershell
java -jar .\sonar-cnes-report-5.0.4.jar --help
```

The key options required for the Security Shepherd use case are:

| Option | Description |
|--------|-------------|
| `-s, --server` | Complete URL of the SonarQube server |
| `-p, --project` | SonarQube project key of the targeted project |
| `-t, --token` | SonarQube authentication token |
| `-o, --output` | Output directory for generated report files |
| `-a, --author` | Name of the report author (optional) |
| `-l, --language` | Report language: `en_US` or `fr_FR` (optional, default: `en_US`) |

Additional options available but not used in this example:

```text
-b, --branch        Branch of the targeted project
-c, --disable-conf  Disable export of quality configuration
-d, --date          Date for the report (format: yyyy-MM-dd)
-e, --disable-spreadsheet
-f, --disable-csv
-m, --disable-markdown
-w, --disable-report
-x, --template-spreadsheet
-r, --template-report
-n, --template-markdown
-v, --version
```

---

# 71. CNES Report — Security Shepherd Configuration

The values used for the Security Shepherd project in the test environment:

| Parameter | Generic Placeholder | Local/Lab Value Used |
|-----------|--------------------|----------------------|
| SonarQube server | `<SONARQUBE_URL>` | `http://localhost:9000` |
| SonarQube project key | `Security-Shepherd` | `Security-Shepherd` |
| SonarQube project name | `Security Shepherd` | `Security Shepherd` |
| CNES JAR directory | `<CNES_REPORT_DIR>` | `V:\platform-tools\CNES-Report` |
| CNES output directory | `<CNES_OUTPUT_DIR>` | `.\Security-Shepherd-report` |
| Report author | `<REPORT_AUTHOR>` | `Vasanth` |
| Report language | `en_US` | `en_US` |

The project key `Security-Shepherd` must match exactly the project key configured in SonarQube.

The SonarQube server must be running and the Security Shepherd project must already have a completed SonarQube analysis before generating the CNES report.

---

# 72. Create the CNES Output Directory

Navigate to the directory where the CNES JAR is stored:

```powershell
cd <CNES_REPORT_DIR>
```

> **Environment-specific example from the test environment:**
> The original implementation used:
> `cd V:\platform-tools\CNES-Report`
> Use any directory on your machine that contains the CNES JAR.

Create the output directory:

```powershell
New-Item -ItemType Directory -Force .\Security-Shepherd-report
```

The `-Force` flag means this command is harmless if the directory already exists.

---

# 73. CNES Report Command

Run the following PowerShell command from `<CNES_REPORT_DIR>` (the directory containing the CNES JAR):

```powershell
java -jar .\sonar-cnes-report-5.0.4.jar `
  -s "<SONARQUBE_URL>" `
  -p "Security-Shepherd" `
  -t "<SONARQUBE_TOKEN>" `
  -o ".\Security-Shepherd-report" `
  -a "<REPORT_AUTHOR>" `
  -l "en_US"
```

Equivalent one-line form:

```powershell
java -jar .\sonar-cnes-report-5.0.4.jar -s "<SONARQUBE_URL>" -p "Security-Shepherd" -t "<SONARQUBE_TOKEN>" -o ".\Security-Shepherd-report" -a "<REPORT_AUTHOR>" -l "en_US"
```

**Example from the test environment** (local lab, localhost SonarQube, author Vasanth):

```powershell
java -jar .\sonar-cnes-report-5.0.4.jar -s "http://localhost:9000" -p "Security-Shepherd" -t "<SONARQUBE_TOKEN>" -o ".\Security-Shepherd-report" -a "Vasanth" -l "en_US"
```

> **IMPORTANT:** Replace `<SONARQUBE_TOKEN>` with your actual SonarQube token when running the command locally.
> **Never commit the real token to Git or include it in any documentation.**
> Replace `<SONARQUBE_URL>` with the URL of your SonarQube server (e.g. `http://localhost:9000` for a local setup).
> Replace `<REPORT_AUTHOR>` with your name or leave the `-a` flag out entirely.

---

# 74. CNES Command Arguments Explained

```text
-s "<SONARQUBE_URL>"
    SonarQube server URL.
    Local/lab example: http://localhost:9000
    Replace with the correct URL if SonarQube is on a different host.

-p "Security-Shepherd"
    SonarQube project key.
    Must match the project key configured in SonarQube exactly.
    For this implementation: Security-Shepherd

-t "<SONARQUBE_TOKEN>"
    SonarQube authentication token.
    The token must have permission to access the Security-Shepherd project.
    Do NOT store the real token in source control.
    Replace with your token at runtime only.

-o ".\Security-Shepherd-report"
    Output directory for generated report files.
    This is a path relative to <CNES_REPORT_DIR>.
    Environment-specific full path example: <CNES_REPORT_DIR>\Security-Shepherd-report

-a "<REPORT_AUTHOR>"
    Report author name.
    Optional. Included in the generated report.
    Example value used in test environment: Vasanth

-l "en_US"
    Report language.
    Optional. Defaults to en_US.
```

---

# 75. What the CNES Command Does

When executed, CNES Report performs the following steps:

```text
1. SonarQube must be running at http://localhost:9000
2. Security Shepherd must already have a completed SonarQube analysis
3. CNES connects to the SonarQube server
4. CNES authenticates using the supplied token
5. CNES accesses the Security-Shepherd project by project key
6. CNES exports the available SonarQube analysis results
7. CNES generates report files in Security-Shepherd-report\
```

Architecture:

```text
SonarQube
    |
    | Security-Shepherd project
    |
    v
CNES Report JAR
    |
    v
Security-Shepherd-report\
    |
    +--> generated report files
```

CNES is a reporting and export utility. It reads existing SonarQube results and produces standalone output files. It does not perform any analysis itself.

---

# 76. CNES Output Location

The CNES JAR is run from `<CNES_REPORT_DIR>` (the directory where the JAR is stored).

The output option used is:

```text
-o ".\Security-Shepherd-report"
```

This places the generated files in a subdirectory relative to `<CNES_REPORT_DIR>`:

```text
<CNES_OUTPUT_DIR>
    |
    +--> generated report files
```

> **Environment-specific example from the test environment:**
> The JAR was run from `V:\platform-tools\CNES-Report`
> The output was placed in `V:\platform-tools\CNES-Report\Security-Shepherd-report\`
> For your setup, choose any suitable directory.

CNES can generate multiple output formats. The CNES help output indicates that report generation can include:

```text
- Report (document format)
- Spreadsheet
- CSV
- Markdown
- Quality configuration export
```

Check the generated output directory after the command completes to see the files that were produced.

---

# 77. CNES Is NOT Part of the Jenkins Pipeline

To be explicit:

```text
CNES Report is NOT configured in the current Jenkinsfile.
```

The Jenkinsfile currently generates and archives:

```text
- Dependency-Check XML report
- Dependency-Check HTML report
- JUnit Surefire test reports
- WAR artifact
```

SonarQube analysis results are sent to the SonarQube server and are viewed through the SonarQube web dashboard.

CNES is currently a separate, manual step run locally after SonarQube analysis.

The complete current flow is:

```text
Jenkins
    |
    +--> Dependency-Check
    |       |
    |       +--> XML report (Jenkins artifact)
    |       +--> HTML report (Jenkins artifact)
    |
    +--> SonarQube
            |
            +--> SonarQube Dashboard    <-- view SAST results here
                    |
                    +--> OPTIONAL CNES Report   <-- manual, local step
                            |
                            +--> Standalone exported reports
```

---

# 78. CNES Security Warning

The CNES command requires a SonarQube authentication token.

Never:

```text
- commit the token to Git
- put the real token in SECURITY_CI_CD.md
- put the real token in Jenkinsfile
- share the token publicly
- store the token in shell scripts committed to the repository
```

Always use a placeholder such as:

```text
<SONARQUBE_TOKEN>
```

in documentation. Supply the real token only at runtime, locally.

---

# 79. CNES Local/Lab Configuration Note

The SonarQube URL `http://localhost:9000` is the **local/lab** SonarQube URL used in this implementation.

For a different machine or production deployment, replace it with the appropriate SonarQube server URL.

The `<CNES_REPORT_DIR>` and `<CNES_OUTPUT_DIR>` placeholders in the commands represent directories that are specific to each user's machine:

```text
<CNES_REPORT_DIR>
    The directory where sonar-cnes-report-5.0.4.jar is stored.
    Environment-specific example: V:\platform-tools\CNES-Report

<CNES_OUTPUT_DIR>
    The directory where CNES will write report files.
    Environment-specific example: V:\platform-tools\CNES-Report\Security-Shepherd-report
```

For a different environment, choose any suitable directories.

---

# 80. Post-Build Actions

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

# 81. Archived Test Reports

Jenkins archives:

```text
target/surefire-reports/**
```

These files contain the JUnit/Surefire test results.

---

# 82. Archived Dependency-Check Reports

Jenkins archives:

```text
dependency-check-report.xml
dependency-check-report.html
```

The HTML file provides a human-readable security report.

The XML file is useful for Jenkins report processing.

---

# 83. Archived WAR

The pipeline also archives:

```text
target/*.war
```

This allows the generated application WAR to be retained with the Jenkins build.

---

# 84. Scanner Report Task File

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

# 85. Build Retention

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

# 86. Final Pipeline Flow

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

# 87. Successful Jenkins Build

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

# 88. Jenkins SCA Report Results

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

# 89. Dependency-Check NVD Status

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

# 90. Dependency-Check Network Warning

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

# 91. Important Interpretation of the SCA Warning

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

# 92. Dependency-Check Local vs Jenkins Execution

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

# 93. Recommended Improvement for Dependency-Check

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

# 94. Recommended Production SCA Architecture

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

# 95. Why Suppressions Were Not Added

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

# 96. Why Severity Thresholds Were Not Added Initially

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

# 97. Recommended Future Security Gate

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

# 98. Troubleshooting — SonarQube Not Running

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

# 99. Troubleshooting — SonarQube URL

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

# 100. Troubleshooting — SonarQube Token

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

# 101. Troubleshooting — SonarQube Prefix Error

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

# 102. Troubleshooting — Quality Gate Waiting

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

# 103. Troubleshooting — Dependency-Check NVD Failure

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

# 104. Troubleshooting — Dependency-Check External Sources

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

# 105. Troubleshooting — Dependency-Check Report Missing

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

# 106. Troubleshooting — Dependency-Check Installation

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

# 107. Troubleshooting — NVD Credential

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

# 108. Troubleshooting — Maven Not Found

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

# 109. Troubleshooting — Java Not Found

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

# 110. Troubleshooting — Windows Docker Build

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

# 111. Troubleshooting — Empty JUnit Results

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

# 112. Jenkins Build Failure Investigation

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

# 113. Understanding the Final Jenkins Result

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

# 114. Security Shepherd Is Intentionally Vulnerable

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

# 115. Example SonarQube Interpretation

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

# 116. Example SCA Interpretation

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

# 117. Security Report Lifecycle

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

# 118. What Is Automated vs Manual

## Automated by Jenkins

The following are fully automated by the Jenkins pipeline on every build:

### Build

```text
mvn clean package -B -DskipTests
```

### Unit Tests

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
JUnit publisher
Dependency-Check publisher
SonarQube dashboard (via webhook)
```

### Artifact archiving

```text
Jenkins archiveArtifacts (post block)
```

## Manual / Optional

### CNES SonarQube Report

CNES report generation is **NOT automated**.

It is a separate, manual, local step that can be run after a SonarQube analysis is complete.

CNES consumes existing SonarQube project results and generates standalone exportable report files. It does not perform any analysis itself.

See sections 68–79 for the complete CNES setup and command.

---

# 119. Secrets Management

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

# 120. Current Jenkins Configuration Summary

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

# 121. Current SonarQube Configuration Summary

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

# 122. Current Dependency-Check Configuration Summary

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

# 123. Current Test Results

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

# 124. Current SCA Results

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

# 125. Current SAST Results

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

# 126. What This Project Demonstrates

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

# 127. Why Both SAST and SCA Are Used

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

# 128. Why Jenkins Is Used

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

# 129. Reproducing the Setup on Another Machine

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

# 130. Production Considerations

The current implementation is primarily a local/lab DevSecOps setup.

Before using it as a production CI/CD security gate, consider the following improvements.

## 130.1 Use a reachable SonarQube/Jenkins URL

Instead of:

```text
http://localhost:8080/sonarqube-webhook/
```

use a properly reachable Jenkins endpoint.

---

## 130.2 Persist Dependency-Check Data

Use persistent storage for the Dependency-Check vulnerability database.

---

## 130.3 Introduce Security Thresholds

After establishing a baseline, introduce policies such as:

```text
Fail on newly introduced Critical vulnerabilities
```

or:

```text
Fail on newly introduced High/Critical vulnerabilities
```

---

## 130.4 Manage False Positives Properly

Use documented Dependency-Check suppressions only when justified.

Do not suppress findings merely to achieve a green build.

---

## 130.5 Secure Jenkins

Use:

- HTTPS
- Authentication
- Authorization
- Least-privilege permissions
- Secure credentials
- Restricted agent access
- Regular plugin updates

---

## 130.6 Secure SonarQube

Use:

- HTTPS
- Authentication
- Appropriate project permissions
- Secure token management
- Controlled webhook access

---

# 131. Future Improvements

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

# 132. Recommended Mature DevSecOps Pipeline

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

# 133. Security Principles Followed

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

# 134. Important Limitations

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

# 135. Final Result

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

# 136. Quick Reference

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

## SonarQube Project Name

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

Generic (replace with your SonarQube server address):

```text
<SONARQUBE_URL>
```

Local/lab value used in this implementation:

```text
http://localhost:9000
```

## SonarQube Jenkins Credential

```text
sonarqube-token
```

(Credential identifier — not the actual token value)

## SonarQube Webhook

Generic:

```text
<JENKINS_URL>/sonarqube-webhook/
```

Local/lab value used in this implementation:

```text
http://localhost:8080/sonarqube-webhook/
```

## Dependency-Check Installation Name

```text
OWASP-DC
```

## NVD Credential

```text
NVD_API_KEY
```

(Credential identifier — not the actual API key value)

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

## CNES Report (OPTIONAL — manual local step)

```text
CNES JAR:     sonar-cnes-report-5.0.4.jar
CNES Server:  <SONARQUBE_URL>
CNES Project: Security-Shepherd
CNES Output:  <CNES_OUTPUT_DIR>
```

Local/lab values used in this implementation:

```text
CNES Server:  http://localhost:9000
CNES Output:  <CNES_REPORT_DIR>\Security-Shepherd-report\
```

CNES is NOT part of the automated Jenkins pipeline. Run manually after a completed SonarQube analysis when a standalone exportable report is required.

---

# 137. Conclusion

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
