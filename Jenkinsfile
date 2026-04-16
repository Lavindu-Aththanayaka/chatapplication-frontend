pipeline {
    agent any

    environment {
        // WSL/Linux-friendly PATH. Remove macOS Homebrew path.
        PATH = "/usr/local/bin:/usr/bin:/bin:${env.PATH}"

        DOCKER_HOST = "unix:///var/run/docker.sock"
    }

    stages {
        stage('Check Docker Installation on Jenkins (WSL)') {
            steps {
                script {
                    if (sh(script: 'command -v docker >/dev/null 2>&1', returnStatus: true) == 0) {
                        echo 'Docker CLI is available in Jenkins WSL environment.'
                        sh '''
                            set -e
                            docker --version
                            docker info >/dev/null 2>&1 || {
                                echo "Docker daemon is not reachable from Jenkins in WSL."
                                echo "Make sure Docker Desktop integration for WSL is enabled"
                                echo "or that the Jenkins user can access /var/run/docker.sock."
                                exit 1
                            }
                        '''
                    } else {
                        error '''
Docker CLI is not installed in the Jenkins WSL environment.
Install Docker CLI inside WSL, or enable Docker Desktop WSL integration.
'''
                    }
                }
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/Lavindu-Aththanayaka/chatapplication-frontend.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image in WSL...'
                    if (fileExists('Dockerfile')) {
                        sh '''
                            set -e
                            docker build --platform linux/amd64 -t lavindu/frontend1:latest .
                        '''
                    } else {
                        error 'Dockerfile not found in the repository.'
                    }
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ec4dcb1b-cbaa-4f8e-9dc3-6d8616667b49',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                    '''
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    echo 'Pushing Docker image to Docker Hub...'
                    def imageId = sh(
                        script: 'docker images -q lavindu/frontend1:latest',
                        returnStdout: true
                    ).trim()

                    if (imageId) {
                        retry(3) {
                            sh 'docker push lavindu/frontend1:latest'
                        }
                    } else {
                        error 'Docker image not found. Build step might have failed.'
                    }
                }
            }
        }

        stage('Deploy Locally (WSL)') {
            steps {
                sh '''
                    docker stop frontend || true
                    docker rm -f frontend || true
                    docker run -d --name frontend -p 3000:3000 --restart unless-stopped lavindu/frontend1:latest
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}