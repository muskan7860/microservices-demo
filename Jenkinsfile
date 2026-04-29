pipeline {
    agent any

    environment {
        PROJECT = "microservices-demo"
        IMAGE = "muskanpatel71198/microservices-app:v1"
        REGISTRY = "docker.io"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Image (Kaniko)') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {

                    sh '''
                    echo "Creating Docker config for Kaniko"

                    mkdir -p /kaniko/.docker

                    echo "{\"auths\":{\"https://index.docker.io/v1/\":{\"username\":\"$DOCKER_USER\",\"password\":\"$DOCKER_PASS\"}}}" > /kaniko/.docker/config.json

                    /kaniko/executor \
                        --context `pwd` \
                        --dockerfile Dockerfile \
                        --destination $IMAGE
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl apply -f release/kubernetes-manifests.yaml
                kubectl rollout status deployment frontend -n dev
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline Success! CI/CD Completed"
        }
        failure {
            echo "❌ Pipeline Failed! Check logs"
        }
    }
}
