pipeline {
    agent any

    environment {
        IMAGE = "muskanpatel71198/microservices-app:v1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS')]) {

                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $IMAGE
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
}
