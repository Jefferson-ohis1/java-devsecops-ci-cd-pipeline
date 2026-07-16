pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        IMAGE_NAME = 'jeffersonohis1/my-java-app'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('RunSonarCloudAnalysis') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    sh 'mvn clean verify sonar:sonar -Dsonar.login=$SONAR_TOKEN -Dsonar.organization=jefferson-ohis1-org -Dsonar.host.url=https://sonarcloud.io -Dsonar.projectKey=jefferson-ohis1-org_java-apps'

                }
            }
        }

        stage('Build Java Application') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('Snyk Scan') {
            tools {
                jdk 'jdk17'
                maven 'maven3'
            }

            environment {
                SNYK_TOKEN = credentials('SNYK_TOKEN')
            }

            steps {
                dir("${WORKSPACE}") {
                    sh '''
                    curl -Lo snyk https://static.snyk.io/cli/latest/snyk-linux
                    chmod +x snyk
                    chmod +x mvnw

                    ./snyk auth --auth-type=token $SNYK_TOKEN
                    
                    ./mvnw dependency:tree -DoutputType=dot

                    ./snyk test \
                        --all-projects \
                        --severity-threshold=medium \
                        --json-file-output=snyk-report.json || true
                    
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "v${BUILD_NUMBER}"
                }

                sh '''
                     docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                     docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                
                '''
            }
        }

        stage('Trivy Image Scan') {
             steps {
                sh '''
                    export TMPDIR=/var/tmp

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --exit-code 0 \
                        --no-progress \
                        --format table \
                        -o trivy-report.txt \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''
             }
        }

        stage('Run Container') {
            steps {
                sh '''
                docker rm -f devsecops-java-app-container || true

                docker run -d \
                    --name devsecops-java-app-container \
                    -p 8081:8081 \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Wait for Application') {
            steps {
                sh '''
                    echo "Waiting for application to start..."

                    until curl -fs http://localhost:8081/ > /dev/null
                    do
                        sleep 5
                    done
                    
                    echo "Application is running."
                '''
            }
        }

        stage('OWASP ZAP Baseline Scan') {
            steps {
                script {
                    def status = sh(
                        script: '''
                            docker run --rm \
                                --network host \
                                -v $(pwd):/zap/wrk/:rw \
                                ghcr.io/zaproxy/zaproxy:stable \
                                zap-baseline.py \
                                -t http://localhost:8081 \
                                -r zap-report.html
                        ''',
                        returnStatus: true
                    )

                    echo "ZAP exited with code ${status}"

                    switch (status) {
                        case 0:
                            echo "No security issues detected."
                            break

                        case 1:
                            echo "FAIL-level security issues found. Review zap-report.html."
                            break

                        case 2:
                            echo "Warnings found. Review zap-report.html."
                            break

                        case 3:
                            echo "ZAP failed to complete the scan."
                            currentBuild.result = 'UNSTABLE'
                            break

                        default:
                            error("Unexpected ZAP exit code: ${status}")
                    }
                }
            }
        }
            

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'DOCKER_LOGIN',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )
                ]) {
                    sh '''
                        echo $PASSWORD | docker login -u $USERNAME --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }
    }

    post {
         always {
            sh 'docker rm -f devsecops-java-app-container || true'

            archiveArtifacts artifacts: 'trivy-report.txt, zap-report.html, snyk-report.json', fingerprint: true
         }
        success {
            echo '✅ Build and Docker image push successful'
        }

        failure {
            echo '❌ Build failed. Check the logs for details'
        }
    }
}
