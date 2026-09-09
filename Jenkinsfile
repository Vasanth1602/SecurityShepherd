/**
 * Jenkins Declarative Pipeline — OWASP Security Shepherd
 * DevSecOps / SCA + SAST pipeline: compile → unit test → Dependency-Check SCA → SonarQube SAST → Quality Gate → archive
 *
 * Prerequisites (configure in Jenkins before running):
 *   - JDK installation named "JDK-17"      → Manage Jenkins → Tools → JDK
 *   - Maven installation named "Maven-3.9"  → Manage Jenkins → Tools → Maven
 *   - SonarQube server named "SonarQube"    → Manage Jenkins → System → SonarQube servers
 *     (set Server URL + the stored auth token credential there — never here)
 *   - SonarQube webhook for Quality Gate    → SonarQube → Administration → Webhooks
 *     URL: http://<jenkins-host>:<port>/sonarqube-webhook/
 */

pipeline {

    agent any

    // Tool names must match the installations configured in Jenkins → Manage Jenkins → Tools.
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
        timeout(time: 120, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // Jenkins performs the SCM checkout automatically (job is "Pipeline script from SCM").
        // This stage only logs Git context for traceability.
        stage('Checkout') {
            steps {
                echo "Branch : ${env.GIT_BRANCH}"
                echo "Commit : ${env.GIT_COMMIT}"
            }
        }

        // Compile main and test sources. No -Pdocker: that profile runs a Linux shell script
        // (docker/scripts/convert-sql-scripts.sh) to prepare Docker artefacts — it is not
        // needed for compilation or SAST and cannot execute on Windows.
        // -DskipTests skips Surefire execution but still compiles test sources, so the
        // Unit Tests stage reuses those classes without recompiling from scratch.
        // 'package' produces target/owaspSecurityShepherd.war; it does not reach
        // the integration-test or verify phases, so Failsafe and Spotless remain inert.
        stage('Build') {
            steps {
                bat 'mvn clean package -B -DskipTests'
            }
        }

        // Run Surefire unit tests (src/test/java). Integration tests (src/it/java) are not
        // executed here — they require a live MariaDB/MongoDB stack and the native libargon2
        // library. Failsafe's integration-test phase is not reached by 'mvn test', so no
        // extra skip flag is needed.
        // Note: pom.xml Surefire already sets -Duser.timezone=UTC in <argLine>.
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

        // Software Composition Analysis (SCA) — checks third-party dependencies for known CVEs.
        // Uses the Jenkins OWASP Dependency-Check plugin with the OWASP-DC tool installation.
        // NVD API key is injected from Jenkins credentials (NVD_API_KEY) — never hardcoded here.
        // Reports are written to the workspace root:
        //   dependency-check-report.xml  (consumed by dependencyCheckPublisher)
        //   dependency-check-report.html (human-readable; archived as a build artifact)
        // First integration: reporting-only baseline. No failure thresholds are applied yet.
        // Vulnerability thresholds (--failOnCVSS / publisher rules) will be added after
        // reviewing the baseline scan results.
        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--project "Security Shepherd" --format XML --format HTML',
                                nvdCredentialsId: 'NVD_API_KEY',
                                odcInstallation: 'OWASP-DC'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        // Static application security testing (SAST) via SonarQube Maven scanner.
        // withSonarQubeEnv injects SONAR_HOST_URL and the auth token from the Jenkins-managed
        // SonarQube server credential — no token appears in this file.
        // 'SonarQube' must match the server name in Manage Jenkins → System → SonarQube servers.
        //
        // Exclusions:
        //   mobile/**                        — Android Gradle project, not part of Maven build
        //   src/main/resources/database/**   — SQL schema files; SonarQube Community has no SQL
        //                                      analyser, so including them adds noise only
        //
        // The intentionally vulnerable Java source under src/main/java/servlets/module/ is
        // NOT excluded — detecting those vulnerabilities is the purpose of this SAST scan.
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    // Full plugin coordinates required — the 'sonar:' prefix shorthand
                    // was retired. Version pinned to latest stable (May 2026).
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

        // Wait for SonarQube to finish the analysis task and return the Quality Gate result.
        // abortPipeline:true fails the build if the gate is not OK.
        // Requires the SonarQube webhook configured above; without it this stage will timeout.
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
            // Archived artifacts:
            //   target/surefire-reports/**           — JUnit unit-test results
            //   dependency-check-report.xml          — DC SCA machine-readable report (used by publisher)
            //   dependency-check-report.html         — DC SCA human-readable report
            //   .scannerwork/report-task.txt         — SonarQube scanner metadata (ceTaskId, dashboardUrl)
            //                                          sonar-maven-plugin 5.x writes here; check
            //                                          '[INFO] Working dir:' in the build log to confirm
            //   target/*.war                         — compiled application artefact
            archiveArtifacts(
                artifacts: 'target/surefire-reports/**, dependency-check-report.xml, dependency-check-report.html, .scannerwork/report-task.txt, target/*.war',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }
        success  { echo 'Pipeline succeeded.' }
        unstable { echo 'Pipeline unstable — check test results or Quality Gate.' }
        failure  { echo 'Pipeline failed — review stage logs above.' }
    }
}
