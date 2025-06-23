pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube-10' // Nom du serveur dans Jenkins > Configure System
        DOCKER_IMAGE = "hasnahatti70/spring-boot-testing"
        BUILD_VERSION = readMavenPom().getVersion()
    }

    stages {
        stage('Clean Workspace') {
            steps {
                script {
                    cleanWs()
                }
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/hasnahatti70/spring-boot-testing.git'
            }
        }

        stage('Gitleaks Scan') {
            steps {
                bat '''
                echo 🔍 Analyse avec Gitleaks...
                "C:\\Users\\MTechno\\Downloads\\gitleaks_8.26.0_windows_x64\\gitleaks.exe" detect --source=. --verbose --report-format=json --report-path=gitleaks-report.json || exit /b 0

                echo 📄 Rapport Gitleaks :
                type gitleaks-report.json || echo ⚠️ Aucun secret détecté.
                '''
            }
        }

        stage('Build & Test') {
            steps {
                bat 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    bat '''
                    mvn verify sonar:sonar ^
                      -Dsonar.projectKey=spring-boot-testing ^
                      -Dsonar.projectName="Spring Boot Testing" ^
                      -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Publish Test Results') {
            steps {
                junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '''
                    -o './' -s './' -f 'ALL' --prettyPrint
                ''', odcInstallation: 'OWASP Dependency Check'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . || true'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE}:${BUILD_VERSION} ."
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerHubUsername', passwordVariable: 'dockerHubPassword')]) {
                        sh "echo ${dockerHubPassword} | docker login -u ${dockerHubUsername} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}:${BUILD_VERSION}"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_IMAGE}:${BUILD_VERSION} || true"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline exécutée avec succès 🚀"
        }
        failure {
            echo "❌ Échec de la pipeline. Consulte les logs Jenkins."
        }
        always {
            archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
        }
    }
}
