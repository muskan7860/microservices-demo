pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    command:
    - cat
    tty: true
"""
        }
    }

    environment {
        IMAGE = "muskanpatel71198/microservices-app:v1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Image') {
            steps {
                container('kaniko') {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS')]) {

                        sh '''
                        echo "{\"auths\":{\"https://index.docker.io/v1/\":{\"username\":\"$USER\",\"password\":\"$PASS\"}}}" > /kaniko/.docker/config.json

                        /kaniko/executor \
                          --context $WORKSPACE \
                          --dockerfile Dockerfile \
                          --destination $IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh 'kubectl apply -f release/kubernetes-manifests.yaml'
            }
        }
    }
}
