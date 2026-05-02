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

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --noupdate', odcInstallation: 'dependency-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

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
                        def context = (service == "cartservice") 
                            ? "./src/cartservice/src" 
                            : "./src/${service}"

                        // ✅ Groovy comments OUTSIDE sh block = OK
                        sh """
                        echo "🔨 Building ${service} from context: ${context}"
                        docker build -t ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} ${context}
                        
                        # ✅ Shell comments INSIDE sh block = use #
                        echo "📋 Verifying image exists:"
                        docker images | grep ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} || echo "⚠️ Image not found locally!"
                        """
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
                        echo "🔍 Scanning ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        trivy image --severity HIGH,CRITICAL --offline-scan ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}
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
                    sh "echo \$PASS | docker login -u \$USER --password-stdin"

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
        // UPDATE K8s YAML
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

                    git push https://\${USER}:\${PASS}@github.com/muskan7860/microservices-demo.git main
                    """
                }
            }
        }
    }
}
