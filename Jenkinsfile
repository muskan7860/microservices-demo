pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "muskanpatel71198/microservices-app:v1"
    }

    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }

        stage('Docker Login & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS')]) {

                    sh '''
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push $DOCKER_IMAGE
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f release/kubernetes-manifests.yaml'
            }
        }
    }

    post {
        success {
            echo "🚀 Pipeline Success! App deployed"
        }
        failure {
            echo "❌ Pipeline Failed! Check logs"
        }
    }
}
