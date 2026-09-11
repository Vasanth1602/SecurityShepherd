/**
 * Jenkins Declarative Pipeline — OWASP Security Shepherd
 * DevSecOps pipeline with SCA (OWASP Dependency-Check) and SAST (SonarQube).
 *
 * Configured Jenkins tools / prerequisites:
 *   - JDK: 'JDK-17'
 *   - Maven: 'Maven-3.9'
 *   - Dependency-Check: 'OWASP-DC' with credential 'NVD_API_KEY'
 *   - SonarQube Server: 'SonarQube' with webhook back to Jenkins
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
        timeout(time: 120, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // Log Git revision information for build traceability
        stage('Checkout') {
            steps {
                echo "Branch : ${env.GIT_BRANCH}"
                echo "Commit : ${env.GIT_COMMIT}"
            }
        }

        // Package WAR and compile test classes without running tests
        stage('Build') {
            steps {
                bat 'mvn clean package -B -DskipTests'
            }
        }

        // Execute unit tests and record JUnit results
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

        // OWASP Dependency-Check SCA — scan third-party dependencies for CVEs
        // Uses persistent local data directory and cached NVD data; NVD API key provided via Jenkins credentials
        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--project "Security Shepherd" --format XML --format HTML --data "%JENKINS_HOME%\\dependency-check-data" --noupdate',
                                nvdCredentialsId: 'NVD_API_KEY',
                                odcInstallation: 'OWASP-DC'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        // SonarQube SAST analysis — server and authentication token are managed by Jenkins configuration
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

        // Enforce SonarQube Quality Gate via webhook callback
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
            // Archive test results, SCA/SAST reports, and packaged WAR
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
