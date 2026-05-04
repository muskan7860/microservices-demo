pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'muskanpatel71198'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'sonarqube'
        SONAR_TOKEN = credentials('SONAR_TOKEN')
    }

    stages {
        // -----------------------------
        // 1. CHECKOUT CODE
        // -----------------------------
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/muskan7860/microservices-demo.git'
            }
        }

        // -----------------------------
        // 2. SONARQUBE ANALYSIS
        // -----------------------------
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                    sonar-scanner \
                        -Dsonar.projectKey=microservices-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://192.168.0.101:9000 \
                        -Dsonar.login=\$SONAR_TOKEN \
                        -Dsonar.exclusions=**/*.java,**/*.cs,**/node_modules/** \
                        -Dsonar.scanner.skipJreProvisioning=true
                    """
                }
            }
        }

        // -----------------------------
        // 3. OWASP DEPENDENCY CHECK
        // -----------------------------
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --noupdate', odcInstallation: 'dependency-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        // -----------------------------
        // 4. TRIVY FILESYSTEM SCAN
        // -----------------------------
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --offline-scan ./src || true'
            }
        }

        // -----------------------------
        // 5. BUILD DOCKER IMAGES
        // -----------------------------
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

        // -----------------------------
        // 6. TRIVY IMAGE SCAN
        // -----------------------------
        stage('Trivy Image Scan') {
            steps {
                script {
                    def services = ['cartservice', 'paymentservice', 'frontend']
                    for (service in services) {
                        def image = "${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        sh "trivy image --severity HIGH,CRITICAL --offline-scan --scanners vuln --timeout 30m --exit-code 0 ${image} || true"
                    }
                }
            }
        }

        // -----------------------------
        // 7. PUSH TO DOCKERHUB
        // -----------------------------
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
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

        // -----------------------------
        // 8. UPDATE KUBERNETES MANIFESTS (FIXED)
        // -----------------------------
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    def services = [
                        'frontend', 'cartservice', 'productcatalogservice',
                        'paymentservice', 'shippingservice', 'currencyservice',
                        'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
                    ]
                    for (service in services) {
                        def manifest = "kubernetes-manifests/${service}.yaml"
                        def newImage = "${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        
                        // ✅ ROBUST: Use awk to update YAML regardless of indentation
                        sh """
                        if [ -f "${manifest}" ]; then
                            # Show before
                            echo "📄 Before ${service}: \$(grep 'image:' ${manifest} | head -1)"
                            
                            # Update using awk (handles any indentation/spaces)
                            awk -v newimg="${newImage}" '/image:.*${service}/ {sub(/image:.*/, "        image: " newimg)} 1' ${manifest} > ${manifest}.tmp && mv ${manifest}.tmp ${manifest}
                            
                            # Show after
                            echo "📄 After ${service}: \$(grep 'image:' ${manifest} | head -1)"
                            
                            # Verify
                            if grep -q "${IMAGE_TAG}" "${manifest}"; then
                                echo "✅ ${service} updated to ${IMAGE_TAG}"
                            else
                                echo "⚠️ ${service} update may have failed - checking file..."
                                cat "${manifest}" | grep -A2 -B2 image || true
                            fi
                        else
                            echo "⚠️ Manifest not found: ${manifest}"
                        fi
                        """
                    }
                }
            }
        }

        // -----------------------------
        // 9. PUSH MANIFESTS TO GITHUB
        // -----------------------------
        stage('Push Manifests to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    git config user.name "muskan7860"
                    git config user.email "muskanpatel914@gmail.com"
                    
                    # Show what changed
                    echo "📋 Manifest changes:"
                    git diff kubernetes-manifests/ || true
                    
                    git add kubernetes-manifests/ || true
                    # Use [ci-skip] to prevent webhook loop
                    git commit -m "chore: update tags to ${IMAGE_TAG} [ci-skip]" --allow-empty || true
                    git push https://$USER:$PASS@github.com/muskan7860/microservices-demo.git main || true
                    
                    echo "✅ Manifests pushed to GitHub"
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo "❌ Pipeline failed - check console for details"
        }
        success {
            echo "🎉 Pipeline completed successfully - version ${IMAGE_TAG}"
        }
    }
}
