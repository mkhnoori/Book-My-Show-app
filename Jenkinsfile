// this file is for deployment to k8s cluster locally
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'mkhnoori1/bms'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/mkhnoori/Book-My-Show-app.git'
                sh 'ls -la'  // Verify files after checkout
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh ''' 
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline failed due to Quality Gate failure: ${qg.status}"
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                ls -la  # Verify package.json exists
                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json  # Remove old dependencies
                    npm install  # Install fresh dependencies
                else
                    echo "Error: package.json not found in bookmyshow-app!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerTag = "${BUILD_NUMBER}"  // Dynamic tag based on BUILD_NUMBER
                    sh "docker build -t ${DOCKER_IMAGE}:${dockerTag} -f bookmyshow-app/Dockerfile bookmyshow-app/"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            echo "${DOCKER_PASS}" | docker login -u ${DOCKER_USER} --password-stdin
                            docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        '''
                    }
                }
            }
        }
        // Deploy to Kubernetes local cluster
        stage('SSH Into k8s Server and Deploy') {
            steps {
                script {
                    def remote = [:]
                    remote.name = 'kmaster'
                    remote.host = '192.168.56.11'
                    remote.user = 'vagrant'
                    remote.password = 'vagrant'
                    remote.allowAnyHosts = true

                    // Clone Git Repo and Deploy Kubernetes Configs in One Script Block
                    sshCommand remote: remote, command: '''
                        git clone https://github.com/mkhnoori/Book-My-Show-app.git
                        cd Book-My-Show-app
                        kubectl apply -f deployment.yml
                        kubectl apply -f service.yml
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "Build ${currentBuild.result} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Project: ${env.JOB_NAME} <br/>
                    Build Number: ${env.BUILD_NUMBER} <br/>
                    Build Status: ${currentBuild.result} <br/>
                    URL: ${env.BUILD_URL} <br/>
                """,
                to: 'mkh.noori@gmail.com',
                attachmentsPattern: '*.txt, *.json'
        }
    }
}
