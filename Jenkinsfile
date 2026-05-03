pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'muskanpatel71198'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/muskan7860/microservices-demo.git'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def services = [
                        'frontend', 'cartservice', 'productcatalogservice',
                        'paymentservice', 'shippingservice', 'currencyservice',
                        'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
                    ]
                    
                    for (service in services) {
                        def context = (service == 'cartservice') ? './src/cartservice/src' : "./src/${service}"
                        sh "docker build -t ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} ${context}"
                    }
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                script {
                    def services = ['cartservice', 'paymentservice', 'frontend']
                    for (service in services) {
                        sh '''
                        trivy image \
                            --severity HIGH,CRITICAL \
                            --offline-scan \
                            --scanners vuln \
                            --timeout 30m \
                            --exit-code 0 \
                            muskanpatel71198/''' + "${service}" + ''':''' + "${IMAGE_TAG}" + ''' || true
                        '''
                    }
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    script {
                        def services = [
                            'frontend', 'cartservice', 'productcatalogservice',
                            'paymentservice', 'shippingservice', 'currencyservice',
                            'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
                        ]
                        for (service in services) {
                            sh "docker push ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        }
                    }
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    def services = [
                        'frontend', 'cartservice', 'productcatalogservice',
                        'paymentservice', 'shippingservice', 'currencyservice',
                        'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
                    ]
                    for (service in services) {
                        sh '''
                        sed -i "s|image: muskanpatel71198/''' + "${service}" + ''':.*|image: muskanpatel71198/''' + "${service}" + ''':''' + "${IMAGE_TAG}" + '''|g" kubernetes-manifests/''' + "${service}" + '''.yaml" || true
                        '''
                    }
                }
            }
        }

        stage('Push Manifests to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    git config user.name "muskan7860"
                    git config user.email "muskanpatel914@gmail.com"
                    git add kubernetes-manifests/ || true
                    git commit -m "chore: update tags [ci-skip]" --allow-empty || true
                    git push https://$USER:$PASS@github.com/muskan7860/microservices-demo.git main || true
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
