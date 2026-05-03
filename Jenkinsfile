pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'muskanpatel71198'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'sonarqube'
        SONAR_TOKEN = credentials('SONAR_TOKEN')
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
                        def context = (service == 'cartservice') 
                            ? './src/cartservice/src' 
                            : "./src/${service}"
                        
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
                        sh """
                        trivy image \\
                            --severity HIGH,CRITICAL \\
                            --offline-scan \\
                            --scanners vuln \\
                            --timeout 30m \\
                            --exit-code 0 \\
                            ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG} || true
                        """
                    }
                }
            }
        }

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

        stage('Update Kubernetes Manifests') {
          steps {
            script {
              def services = [
                'frontend', 'cartservice', 'productcatalogservice',
                'paymentservice', 'shippingservice', 'currencyservice',
                'emailservice', 'recommendationservice', 'checkoutservice', 'adservice'
              ]
            
              for (service in services) {
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
                        def context = (service == 'cartservice') 
                            ? './src/cartservice/src' 
                            : "./src/${service}"
                        
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
                            ''' + "${DOCKERHUB_REPO}/" + '''${service}:''' + "${IMAGE_TAG}" + ''' || true
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
                        // ✅ Use triple-single-quotes to avoid $ escaping issues
                        sh '''
                        MANIFEST="kubernetes-manifests/''' + "${service}" + '''.yaml"
                        echo "🔍 Updating $MANIFEST"
                        
                        if [ -f "$MANIFEST" ]; then
                            # Show before
                            echo "   Before: $(grep 'image:' "$MANIFEST" 2>/dev/null || echo 'NOT FOUND')"
                            
                            # Update with precise sed pattern
                            sed -i "s|image: ''' + "${DOCKERHUB_REPO}" + '''/''' + "${service}" + ''':.*|image: ''' + "${DOCKERHUB_REPO}" + '''/''' + "${service}" + ''':''' + "${IMAGE_TAG}" + '''|g" "$MANIFEST"
                            
                            # Show after
                            echo "   After: $(grep 'image:' "$MANIFEST" 2>/dev/null || echo 'NOT FOUND')"
                            
                            # Verify
                            if grep -q "''' + "${IMAGE_TAG}" + '''" "$MANIFEST"; then
                                echo "✅ Updated to ''' + "${IMAGE_TAG}" + '''"
                            else
                                echo "❌ Update failed - check pattern"
                            fi
                        else
                            echo "⚠️ Manifest not found: $MANIFEST"
                        fi
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
                    # Add [ci-skip] to prevent webhook loop
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
}        def manifest = "kubernetes-manifests/${service}.yaml"
                
                // ✅ Debug: Show before state
                sh "echo '📄 Before: $(grep \"image:\" ${manifest} 2>/dev/null || echo \"file not found\")'"
                
                // ✅ Precise sed replacement with proper quoting
                sh """
                if [ -f "${manifest}" ]; then
                    sed -i "s|image: ${DOCKERHUB_REPO}/${service}:.*|image: ${DOCKERHUB_REPO}/${service}:${IMAGE_TAG}|g" "${manifest}"
                    echo "✅ Updated ${manifest}"
                else
                    echo "⚠️ Manifest not found: ${manifest}"
                fi
                """
                
                // ✅ Debug: Show after state
                sh "echo '📄 After: $(grep \"image:\" ${manifest} 2>/dev/null || echo \"file not found\")'"
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
                    git commit -m "Updated image tags" --allow-empty || true
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
