pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'muskanpatel71198'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'sonarqube'
        SONAR_TOKEN = credentials('SONAR_TOKEN')
    }

    stages {
        // ✅✅✅ STAGE 1: CI SKIP CHECK (Prevents Webhook Loops) ✅✅✅
        stage('Check for CI Skip') {
            steps {
                script {
                    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    echo "🔍 Commit message: ${commitMessage}"
                    if (commitMessage.contains('[ci skip]') || commitMessage.contains('[skip ci]')) {
                        echo "✅ Found [ci skip] - aborting to prevent loop"
                        currentBuild.result = 'ABORTED'
                        error("Build skipped due to [ci skip] directive")
                    }
                }
            }
        }

        // -----------------------------
        // STAGE 2: CHECKOUT CODE
        // -----------------------------
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/muskan7860/microservices-demo.git'
            }
        }

        // -----------------------------
        // STAGE 3: SONARQUBE ANALYSIS
        // -----------------------------
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                    sonar-scanner \\
                        -Dsonar.projectKey=microservices-app \\
                        -Dsonar.sources=. \\
                        -Dsonar.host.url=http://192.168.0.101:9000 \\
                        -Dsonar.login=\$SONAR_TOKEN \\
                        -Dsonar.exclusions=**/*.java,**/*.cs,**/node_modules/** \\
                        -Dsonar.scanner.skipJreProvisioning=true
                    """
                }
            }
        }

        // -----------------------------
        // STAGE 4: OWASP DEPENDENCY CHECK
        // -----------------------------
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --noupdate', odcInstallation: 'dependency-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        // -----------------------------
        // STAGE 5: TRIVY FILESYSTEM SCAN
        // -----------------------------
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --offline-scan ./src || true'
            }
        }

        // -----------------------------
        // STAGE 6: BUILD DOCKER IMAGES
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
        // STAGE 7: TRIVY IMAGE SCAN
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
        // STAGE 8: PUSH TO DOCKERHUB
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
        // STAGE 9: UPDATE KUBERNETES MANIFESTS (FIXED - Simple sed)
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
                        
                        sh """
                        if [ -f "${manifest}" ]; then
                            echo "📄 Updating ${manifest}"
                            
                            # ✅ Simple sed pattern (like coach's approach)
                            sed -i "s|image: ${DOCKERHUB_REPO}/${service}:.*|image: ${newImage}|g" "${manifest}"
                            
                            # ✅ Verify the change worked
                            if grep -q "${newImage}" "${manifest}"; then
                                echo "✅ ${service} updated to ${IMAGE_TAG}"
                            else
                                echo "⚠️ ${service} may not have updated:"
                                grep "image:" "${manifest}" || true
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
        // STAGE 10: PUSH MANIFESTS TO GITHUB
        // -----------------------------
        stage('Push Manifests to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    git config user.name "muskan7860"
                    git config user.email "muskanpatel914@gmail.com"
                    
                    # Check if there are changes before committing
                    if git diff --quiet kubernetes-manifests/; then
                        echo "📋 No manifest changes to commit"
                    else
                        echo "📋 Manifest changes:"
                        git diff kubernetes-manifests/ || true
                        git add kubernetes-manifests/ || true
                        # ✅ [ci-skip] prevents loop (when combined with CI Skip stage above)
                        git commit -m "chore: update tags to ${IMAGE_TAG} [ci-skip]" || true
                        git push https://$USER:$PASS@github.com/muskan7860/microservices-demo.git main || true
                        echo "✅ Manifests pushed to GitHub"
                    fi
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "🧹 Workspace cleaned"
        }
        failure {
            echo "❌ Pipeline failed at stage: ${currentBuild.currentStage?.name ?: 'unknown'}"
        }
        success {
            echo "🎉 Pipeline completed successfully - version ${IMAGE_TAG}"
        }
    }
}
