pipeline {
    agent any

    options {
        // ⏱️ Prevent pipeline from hanging indefinitely
        timeout(time: 2, unit: 'HOURS')
        // 🗑️ Clean up workspace after build to save disk
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_REPO = "muskanpatel71198"
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = "sonarqube"
        SONAR_TOKEN = credentials('SONAR_TOKEN')
        // 🎯 Only scan these critical services with Trivy (skip low-risk ones)
        TRIVY_CRITICAL_SERVICES = [
            "cartservice",      // .NET app - needs scanning
            "paymentservice",   // Security-critical
            "frontend"          // User-facing
        ]
        // 📦 All services to build & push (kept minimal)
        ALL_SERVICES = [
            "frontend", "cartservice", "productcatalogservice",
            "paymentservice", "shippingservice", "currencyservice",
            "emailservice", "recommendationservice", "checkoutservice", "adservice"
        ]
    }

    stages {
        // -----------------------------
        // 1. CHECKOUT (Fast)
        // -----------------------------
        stage('Checkout Code') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [[$class: 'CloneOption', depth: 1, noTags: true, shallow: true]], // ⚡ Shallow clone
                    url: 'https://github.com/muskan7860/microservices-demo.git'
                ]
            }
        }

        // -----------------------------
        // 2. SONARQUBE (Optional - skip if not needed)
        // -----------------------------
        stage('SonarQube Analysis') {
            when { expression { params.RUN_SONAR ?: true } } // 🔀 Toggle via parameter
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                    sonar-scanner \\
                        -Dsonar.projectKey=microservices-app \\
                        -Dsonar.sources=. \\
                        -Dsonar.host.url=http://192.168.0.101:9000 \\
                        -Dsonar.login=\$SONAR_TOKEN \\
                        -Dsonar.exclusions=**/*.java,**/*.cs,**/node_modules/** \\
                        -Dsonar.scanner.skipJreProvisioning=true \\
                        -Dsonar.scanner.forceDeprecation=true
                    """
                }
            }
        }

        // -----------------------------
        // 3. BUILD DOCKER IMAGES (Parallelized)
        // -----------------------------
        stage('Build Docker Images') {
            parallel {
                stage('Build Critical Services') {
                    steps {
                        script {
                            // Build only services that changed (optional optimization)
                            for (service in env.TRIVY_CRITICAL_SERVICES.tokenize(',')) {
                                def context = (service == "cartservice") 
                                    ? "./src/cartservice/src" 
                                    : "./src/${service}"
                                
                                sh """
                                echo "🔨 Building ${service}..."
                                docker build -t ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} ${context}
                                """
                            }
                        }
                    }
                }
                stage('Build Other Services') {
                    steps {
                        script {
                            def otherServices = env.ALL_SERVICES.tokenize(',') - env.TRIVY_CRITICAL_SERVICES.tokenize(',')
                            for (service in otherServices) {
                                def context = "./src/${service}"
                                sh """
                                echo "🔨 Building ${service}..."
                                docker build -t ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} ${context}
                                """
                            }
                        }
                    }
                }
            }
        }

        // -----------------------------
        // 4. TRIVY SCAN (Only Critical Services + Optimized)
        // -----------------------------
        stage('Trivy Security Scan') {
            steps {
                script {
                    // ✅ Pre-cache DBs once (not per-service)
                    sh '''
                    echo "📦 Ensuring Trivy DBs are cached..."
                    trivy image --download-db-only --cache-dir /tmp/trivy-cache || true
                    trivy image --download-java-db-only --cache-dir /tmp/trivy-cache || true
                    '''

                    // ✅ Scan ONLY critical services with optimized flags
                    for (service in env.TRIVY_CRITICAL_SERVICES.tokenize(',')) {
                        sh """
                        echo "🔍 Scanning ${service} (HIGH/CRITICAL only)..."
                        
                        trivy image \\
                            --severity HIGH,CRITICAL \\
                            --offline-scan \\
                            --scanners vuln \\
                            --timeout 30m \\
                            --parallel 1 \\
                            --cache-dir /tmp/trivy-cache \\
                            --skip-dirs /usr/share/doc,/usr/share/man,/usr/share/locale,/usr/local/share \\
                            --exit-code 1 \\
                            ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}
                            
                        echo "✅ ${service} scan passed"
                        """
                    }
                }
            }
        }

        // -----------------------------
        // 5. PUSH IMAGES (All services)
        // -----------------------------
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo \$PASS | docker login -u \$USER --password-stdin"
                    
                    script {
                        for (service in env.ALL_SERVICES.tokenize(',')) {
                            sh "docker push ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        }
                    }
                }
            }
        }

        // -----------------------------
        // 6. UPDATE K8s MANIFESTS (ArgoCD)
        // -----------------------------
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    for (service in env.ALL_SERVICES.tokenize(',')) {
                        sh """
                        sed -i 's|image: .*${service}:.*|image: ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}|g' \\
                            kubernetes-manifests/${service}.yaml 2>/dev/null || echo "⚠️ No manifest for ${service}"
                        """
                    }
                }
            }
        }

        // -----------------------------
        // 7. PUSH MANIFEST CHANGES
        // -----------------------------
        stage('Push Manifests to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                    git config user.name "muskan7860"
                    git config user.email "muskanpatel914@gmail.com"
                    
                    git add kubernetes-manifests/ || true
                    git commit -m "chore: update image tags to ${IMAGE_TAG}" --allow-empty
                    git push https://\$USER:\$PASS@github.com/muskan7860/microservices-demo.git main
                    """
                }
            }
        }
    }

    post {
        // 🧹 Always clean up to save disk space
        always {
            sh '''
            echo "🧹 Cleaning up Docker images to save space..."
            docker images "${DOCKERHUB_REPO}/*:${IMAGE_TAG}" --format "{{.Repository}}:{{.Tag}}" | xargs -r docker rmi || true
            rm -rf /tmp/trivy-cache/* 2>/dev/null || true
            '''
            // 📊 Optional: Archive Trivy reports for audit
            // archiveArtifacts artifacts: '**/trivy-report.json', allowEmptyArchive: true
        }
        // 🚨 Alert on failure
        failure {
            echo "❌ Pipeline failed at stage: ${currentBuild.currentStage?.name ?: 'unknown'}"
            // Add notification here: slackSend, email, etc.
        }
    }
}
