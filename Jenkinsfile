pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "muskanpatel71198"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = "sonarqube"
        SONAR_TOKEN = credentials('SONAR_TOKEN')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/muskan7860/microservices-demo.git'
            }
        }

        // -----------------------------
        // SONARQUBE SCAN
        // -----------------------------
        stage('SonarQube Analysis') {
          steps {
            withSonarQubeEnv("${SONARQUBE_SERVER}") {
              sh """
              sonar-scanner \
              -Dsonar.projectKey=microservices-app \
              -Dsonar.sources=. \
              -Dsonar.host.url=http://192.168.0.101:9000 \
              -Dsonar.login=$SONAR_TOKEN \
              -Dsonar.exclusions=**/*.java
              """
        }
    }
}

        // -----------------------------
        // OWASP DEPENDENCY CHECK
        // -----------------------------
        stage('OWASP Dependency Check') {
          steps {
            dependencyCheck additionalArguments: '--scan . --noupdate', odcInstallation: 'dependency-check'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
}
        // -----------------------------
        // TRIVY FILE SYSTEM SCAN
        // -----------------------------
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL ./src'
            }
        }

        // -----------------------------
        // BUILD ALL DOCKER IMAGES
        // -----------------------------
        stage('Build Docker Images') {
            steps {
                script {
                    def services = [
                        "frontend",
                        "cartservice/src",
                        "productcatalogservice",
                        "paymentservice",
                        "shippingservice",
                        "currencyservice",
                        "emailservice",
                        "recommendationservice",
                        "checkoutservice",
                        "adservice"
                    ]

                    for (service in services) {  
                      def context = service == "cartservice/src" ? "./src/cartservice/src" : "./src/${service}"
                      def imageName = service.contains("cartservice") ? "cartservice" : service

                      sh """
                      echo "Building ${imageName} from ${context}"
                      docker build -t ${DOCKERHUB_REPO}/${imageName}:${IMAGE_TAG} ${context}
                      """
}
                    }
                }
            }
        }

        // -----------------------------
        // TRIVY IMAGE SCAN
        // -----------------------------
        stage('Trivy Image Scan') {
            steps {
                script {
                    def services = [
                        "frontend",
                        "cartservice",
                        "productcatalogservice",
                        "paymentservice",
                        "shippingservice",
                        "currencyservice",
                        "emailservice",
                        "recommendationservice",
                        "checkoutservice",
                        "adservice"
                    ]

                    for (service in services) {
                        sh """
                        trivy image --severity HIGH,CRITICAL ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        // -----------------------------
        // PUSH TO DOCKERHUB
        // -----------------------------
        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo $PASS | docker login -u $USER --password-stdin"

                    script {
                        def services = [
                            "frontend",
                            "cartservice",
                            "productcatalogservice",
                            "paymentservice",
                            "shippingservice",
                            "currencyservice",
                            "emailservice",
                            "recommendationservice",
                            "checkoutservice",
                            "adservice"
                        ]

                        for (service in services) {
                            sh "docker push ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        }
                    }
                }
            }
        }

        // -----------------------------
        // UPDATE K8s YAML (FOR ARGOCD)
        // -----------------------------
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    def services = [
                        "frontend",
                        "cartservice",
                        "productcatalogservice",
                        "paymentservice",
                        "shippingservice",
                        "currencyservice",
                        "emailservice",
                        "recommendationservice",
                        "checkoutservice",
                        "adservice"
                    ]

                    for (service in services) {
                        sh """
                        sed -i 's|image: .*${service}:.*|image: ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}|g' kubernetes-manifests/${service}.yaml
                        """
                    }
                }
            }
        }

        // -----------------------------
        // PUSH UPDATED YAML TO GITHUB
        // -----------------------------
        stage('Push Manifest Changes') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                    git config user.name "muskan7860"
                    git config user.email "muskanpatel914@gmail.com"

                    git add .
                    git commit -m "Updated image tags to ${IMAGE_TAG}" || echo "No changes"

                    git push https://${USER}:${PASS}@github.com/muskan7860/microservices-demo.git main
                    """
                }
            }
        }
    }
}
