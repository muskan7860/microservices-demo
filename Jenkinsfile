pipeline {
    agent any

    options {
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // ⚠️ Continue pipeline even if Trivy finds vulns (set to false for prod)
        skipStagesAfterUnstable(false)
    }

    environment {
        DOCKERHUB_REPO = 'muskanpatel71198'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'sonarqube'
        SONAR_TOKEN = credentials('SONAR_TOKEN')
        TRIVY_CACHE_DIR = '/tmp/trivy-cache'
        SKIP_DOCS_DIRS = '/usr/share/doc,/usr/share/man,/usr/share/locale,/usr/local/share'
        // 🎯 Set to 1 to FAIL pipeline on vulns, 0 to just report
        TRIVY_EXIT_CODE = '1'
    }

    parameters {
        choice(name: 'TRIVY_SERVICES', choices: ['critical', 'all', 'none'], description: 'Trivy scan scope')
        choice(name: 'TRIVY_MODE', choices: ['fail', 'warn'], description: 'Fail pipeline on vulns or just warn?')
        booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube?')
        booleanParam(name: 'SKIP_PUSH', defaultValue: false, description: 'Skip Docker push?')
    }

    stages {
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

        stage('Build Docker Images') {
            steps {
                script {
                    def allServices = [
                        'frontend', 'cartservice', 'productcatalogservice',
                        'paymentservice', 'shippingservice', 'currencyservice',
                        'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
                    ]
                    
                    for (service in allServices) {
                        def context = (service == 'cartservice') 
                            ? './src/cartservice/src' 
                            : "./src/${service}"
                        
                        sh """
                        echo "🔨 Building ${service}..."
                        docker build -t ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} ${context}
                        """
                    }
                    env.ALL_SERVICES_LIST = allServices.join(',')
                }
            }
        }

        stage('Trivy Security Scan') {
            when { expression { params.TRIVY_SERVICES != 'none' } }
            steps {
                script {
                    def criticalServices = ['cartservice', 'paymentservice', 'frontend']
                    def allServices = env.ALL_SERVICES_LIST?.split(',') ?: []
                    def servicesToScan = (params.TRIVY_SERVICES == 'critical') 
                        ? criticalServices 
                        : (params.TRIVY_SERVICES == 'all' ? allServices : criticalServices)
                    
                    // Set exit code based on parameter
                    def exitCode = (params.TRIVY_MODE == 'fail') ? '1' : '0'
                    
                    sh """
                    mkdir -p ${TRIVY_CACHE_DIR}
                    trivy image --download-db-only --cache-dir ${TRIVY_CACHE_DIR} || true
                    trivy image --download-java-db-only --cache-dir ${TRIVY_CACHE_DIR} || true
                    """
                    
                    for (service in servicesToScan) {
                        // ✅ Save report to file for auditing
                        sh """
                        echo "🔍 Scanning ${service}..."
                        
                        trivy image \\
                            --severity HIGH,CRITICAL \\
                            --offline-scan \\
                            --scanners vuln \\
                            --timeout 30m \\
                            --parallel 1 \\
                            --cache-dir ${TRIVY_CACHE_DIR} \\
                            --skip-dirs '${SKIP_DOCS_DIRS}' \\
                            --format table \\
                            --output trivy-report-${service}.txt \\
                            --exit-code ${exitCode} \\
                            ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} || true
                            
                        # ✅ Also generate JSON for CI integration
                        trivy image \\
                            --severity HIGH,CRITICAL \\
                            --offline-scan \\
                            --scanners vuln \\
                            --cache-dir ${TRIVY_CACHE_DIR} \\
                            --format json \\
                            --output trivy-report-${service}.json \\
                            ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} || true
                            
                        echo "📄 Reports saved: trivy-report-${service}.{txt,json}"
                        """
                    }
                    
                    // ✅ Fail pipeline ONLY if mode=fail AND vulns found
                    if (params.TRIVY_MODE == 'fail') {
                        sh """
                        for service in ${servicesToScan.join(' ')}; do
                            if [ -s trivy-report-\$service.txt ] && grep -q 'Total:.*HIGH\\|CRITICAL' trivy-report-\$service.txt; then
                                echo "❌ Vulnerabilities found in \$service - failing pipeline"
                                exit 1
                            fi
                        done
                        echo "✅ All scans passed or no critical vulns"
                        """
                    } else {
                        echo "⚠️ Trivy ran in WARN mode - vulnerabilities reported but pipeline continues"
                    }
                }
            }
        }

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
            // 📊 Archive Trivy reports for audit trail
            archiveArtifacts artifacts: 'trivy-report-*.txt,trivy-report-*.json', allowEmptyArchive: true, fingerprint: true
        }
        // ✅ FIXED: Use proper syntax for post-block status access
        failure {
            echo "❌ Pipeline FAILED"
            // Optional: Send notification here
            // slackSend color: 'danger', message: "Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed"
        }
        unstable {
            echo "⚠️ Pipeline completed with warnings (check Trivy reports)"
        }
        success {
            echo "✅ Pipeline completed successfully"
        }
    }
}
