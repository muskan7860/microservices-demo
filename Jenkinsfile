pipeline {
    agent any

    options {
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        DOCKERHUB_REPO = 'muskanpatel71198'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'sonarqube'
        SONAR_TOKEN = credentials('SONAR_TOKEN')
        // ✅ Strings only in environment block
        TRIVY_CACHE_DIR = '/tmp/trivy-cache'
        SKIP_DOCS_DIRS = '/usr/share/doc,/usr/share/man,/usr/share/locale,/usr/local/share'
    }

    parameters {
        // ✅ Use parameters for configurable lists
        choice(
            name: 'TRIVY_SERVICES',
            choices: ['critical', 'all', 'none'],
            description: 'Which services to scan with Trivy?'
        )
        booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube?')
        booleanParam(name: 'SKIP_PUSH', defaultValue: false, description: 'Skip Docker push?')
    }

    stages {
        // -----------------------------
        // 1. CHECKOUT (Fast shallow clone)
        // -----------------------------
        stage('Checkout Code') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [[$class: 'CloneOption', depth: 1, noTags: true, shallow: true]],
                    url: 'https://github.com/muskan7860/microservices-demo.git'
                ]
            }
        }

        // -----------------------------
        // 2. SONARQUBE (Optional)
        // -----------------------------
        stage('SonarQube Analysis') {
            when { expression { params.RUN_SONAR } }
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
        // 3. BUILD DOCKER IMAGES
        // -----------------------------
        stage('Build Docker Images') {
            steps {
                script {
                    // ✅ Define arrays in script block (not environment)
                    def allServices = [
                        'frontend', 'cartservice', 'productcatalogservice',
                        'paymentservice', 'shippingservice', 'currencyservice',
                        'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
                    ]
                    
                    def criticalServices = ['cartservice', 'paymentservice', 'frontend']
                    
                    // Build all services
                    for (service in allServices) {
                        def context = (service == 'cartservice') 
                            ? './src/cartservice/src' 
                            : "./src/${service}"
                        
                        sh """
                        echo "🔨 Building ${service}..."
                        docker build -t ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} ${context}
                        """
                    }
                    
                    // Store for later stages
                    env.ALL_SERVICES_LIST = allServices.join(',')
                    env.CRITICAL_SERVICES_LIST = criticalServices.join(',')
                }
            }
        }

        // -----------------------------
        // 4. TRIVY SCAN (Only what's needed)
        // -----------------------------
        stage('Trivy Security Scan') {
            when { expression { params.TRIVY_SERVICES != 'none' } }
            steps {
                script {
                    def criticalServices = ['cartservice', 'paymentservice', 'frontend']
                    def allServices = env.ALL_SERVICES_LIST?.split(',') ?: []
                    
                    // Determine which services to scan
                    def servicesToScan = (params.TRIVY_SERVICES == 'critical') 
                        ? criticalServices 
                        : (params.TRIVY_SERVICES == 'all' ? allServices : criticalServices)
                    
                    // Pre-cache Trivy DBs once
                    sh """
                    mkdir -p ${TRIVY_CACHE_DIR}
                    trivy image --download-db-only --cache-dir ${TRIVY_CACHE_DIR} || true
                    trivy image --download-java-db-only --cache-dir ${TRIVY_CACHE_DIR} || true
                    """
                    
                    // Scan selected services
                    for (service in servicesToScan) {
                        sh """
                        echo "🔍 Scanning ${service} (HIGH/CRITICAL)..."
                        
                        trivy image \\
                            --severity HIGH,CRITICAL \\
                            --offline-scan \\
                            --scanners vuln \\
                            --timeout 30m \\
                            --parallel 1 \\
                            --cache-dir ${TRIVY_CACHE_DIR} \\
                            --skip-dirs '${SKIP_DOCS_DIRS}' \\
                            --exit-code 1 \\
                            ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}
                            
                        echo "✅ ${service} scan passed"
                        """
                    }
                }
            }
        }

        // -----------------------------
        // 5. PUSH TO DOCKERHUB (Optional)
        // -----------------------------
        stage('Push to DockerHub') {
            when { expression { !params.SKIP_PUSH } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    
                    script {
                        def services = env.ALL_SERVICES_LIST?.split(',') ?: []
                        for (service in services) {
                            sh "docker push ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}"
                        }
                    }
                }
            }
        }

        // -----------------------------
        // 6. UPDATE K8s MANIFESTS
        // -----------------------------
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    def services = env.ALL_SERVICES_LIST?.split(',') ?: []
                    for (service in services) {
                        sh """
                        sed -i 's|image: .*${service}:.*|image: ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}|g' \\
                            kubernetes-manifests/${service}.yaml 2>/dev/null || echo "⚠️ No manifest: ${service}"
                        """
                    }
                }
            }
        }

        // -----------------------------
        // 7. PUSH MANIFESTS TO GITHUB
        // -----------------------------
        stage('Push Manifests to GitHub') {
            when { expression { !params.SKIP_PUSH } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                    git config user.name "muskan7860"
                    git config user.email "muskanpatel914@gmail.com"
                    
                    git add kubernetes-manifests/ || true
                    git commit -m "chore: update tags to ${IMAGE_TAG}" --allow-empty
                    git push https://\$USER:\$PASS@github.com/muskan7860/microservices-demo.git main
                    """
                }
            }
        }
    }

    post {
        always {
            sh '''
            echo "🧹 Cleaning up..."
            docker images "${DOCKERHUB_REPO}/*:${IMAGE_TAG}" --format "{{.Repository}}:{{.Tag}}" 2>/dev/null | xargs -r docker rmi || true
            rm -rf /tmp/trivy-cache/* 2>/dev/null || true
            '''
        }
        failure {
            echo "❌ Pipeline failed at: ${currentBuild.currentStage?.name ?: 'unknown'}"
        }
    }
}
