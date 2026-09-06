/**
 * Jenkins Declarative Pipeline — OWASP Security Shepherd
 * DevSecOps / SAST pipeline: compile → unit test → SonarQube SAST → Quality Gate → archive
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
        timeout(time: 60, unit: 'MINUTES')
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

        // Static analysis via SonarQube Maven scanner.
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
            // report-task.txt — scanner metadata: projectKey, dashboardUrl, ceTaskId.
            // sonar-maven-plugin 5.x writes to .scannerwork/ (the scanner working dir).
            // Check the build log line '[INFO] Working dir:' to confirm the exact path
            // for your environment; allowEmptyArchive:true prevents failure if absent.
            // This is NOT a vulnerability report — full findings are in the SonarQube dashboard.
            archiveArtifacts(
                artifacts: 'target/surefire-reports/**, .scannerwork/report-task.txt, target/*.war',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }
        success  { echo 'Pipeline succeeded.' }
        unstable { echo 'Pipeline unstable — check test results or Quality Gate.' }
        failure  { echo 'Pipeline failed — review stage logs above.' }
    }
}
