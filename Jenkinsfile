
pipeline {
    agent any

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        IMAGE_NAME = 'VENGASREE/github-jenkins-kubernetes'
    }

    stages {
        stage('Checkout GitHub Code') {
            steps {
                checkout scm
                powershell 'git log -1 --oneline'
            }
        }

        stage('Verify Tools') {
            steps {
                powershell '''
                    docker --version
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker is unavailable"
                    }

                    kubectl get nodes
                    if ($LASTEXITCODE -ne 0) {
                        throw "Kubernetes connection failed"
                    }
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $tag = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                    docker build -t $tag .
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker build failed"
                    }
                '''
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    powershell '''
                        $env:DOCKER_PASS | docker login --username $env:DOCKER_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker login failed"
                        }

                        $tag = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                        docker push $tag
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker push failed"
                        }

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                powershell '''
                    kubectl apply -f deployment.yaml
                    if ($LASTEXITCODE -ne 0) {
                        throw "Deployment apply failed"
                    }

                    $tag = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"

                    kubectl set image deployment/github-demo web=$tag
                    if ($LASTEXITCODE -ne 0) {
                        throw "Image update failed"
                    }

                    kubectl rollout status deployment/github-demo --timeout=180s
                    if ($LASTEXITCODE -ne 0) {
                        throw "Deployment rollout failed"
                    }

                    kubectl apply -f service.yaml
                    if ($LASTEXITCODE -ne 0) {
                        throw "Service apply failed"
                    }
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                powershell 'kubectl get deployment github-demo'
                powershell 'kubectl get pods -l app=github-demo'
                powershell 'kubectl get service github-demo-service'
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: GitHub code built and deployed to Kubernetes.'
        }

        failure {
            echo 'FAILED: Check Console Output for the failed stage.'
        }
    }
}
