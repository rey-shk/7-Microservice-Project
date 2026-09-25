def getServices() {
    return [
        [name: 'auth-service', path: 'services/auth-service'],
        [name: 'catalog-service', path: 'services/catalog-service'],
        [name: 'inventory-service', path: 'services/inventory-service'],
        [name: 'order-service', path: 'services/order-service'],
        [name: 'payment-service', path: 'services/payment-service'],
        [name: 'notification-service', path: 'services/notification-service'],
        [name: 'analytics-service', path: 'services/analytics-service'],
        [name: 'frontend', path: 'frontend']
    ]
}

pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKER_HUB_USER',
            defaultValue: 'shaik98',
            description: 'Docker Hub username or organization'
        )
        string(
            name: 'IMAGE_TAG',
            defaultValue: "${BUILD_NUMBER}",
            description: 'Docker image tag (defaults to Jenkins build number)'
        )
        choice(
            name: 'TRIVY_SEVERITY',
            choices: ['HIGH,CRITICAL', 'CRITICAL', 'LOW,MEDIUM,HIGH,CRITICAL'],
            description: 'Severity threshold for Trivy vulnerability scans'
        )
        booleanParam(
            name: 'WAIT_FOR_QUALITY_GATE',
            defaultValue: false,
            description: 'Wait for SonarQube Quality Gate status and fail build if gate fails'
        )
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        SONAR_SERVER_NAME     = 'SonarQube'
        SCANNER_HOME          = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    }

    stages {
        // ================================================================
        // 1. Checkout SCM
        // ================================================================
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/rey-shk/7-Microservice-Project']])
            }
        }

        // ================================================================
        // 2. SonarQube Code Analysis
        // ================================================================
        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube static code analysis...'
                withSonarQubeEnv("${env.SONAR_SERVER_NAME}") {
                    sh """
                        if [ -d "${env.SCANNER_HOME}/bin" ]; then
                            SCANNER_BIN="${env.SCANNER_HOME}/bin/sonar-scanner"
                        else
                            SCANNER_BIN="sonar-scanner"
                        fi

                        \${SCANNER_BIN} \
                            -Dsonar.projectKey=devopsshack-polyglot-microservices \
                            -Dsonar.projectName="DevOps Shack Polyglot Microservices" \
                            -Dsonar.sources=. \
                            -Dsonar.java.binaries=. \
                            -Dsonar.exclusions="**/node_modules/**,**/target/**,**/.venv/**,**/bin/**,**/obj/**,**/.bundle/**,**/dist/**,**/.git/**,logs/**"
                    """
                }
            }
        }

        // ================================================================
        // 3. SonarQube Quality Gate (Optional)
        // ================================================================
        stage('SonarQube Quality Gate') {
            when {
                expression { params.WAIT_FOR_QUALITY_GATE == true }
            }
            steps {
                echo 'Waiting for SonarQube Quality Gate decision...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ================================================================
        // 4. Trivy Filesystem Scan
        // ================================================================
        stage('Trivy Filesystem Scan') {
            steps {
                echo "Running Trivy filesystem scan (Severity: ${params.TRIVY_SEVERITY})..."
                sh """
                    if command -v trivy >/dev/null 2>&1; then
                        trivy fs \
                            --exit-code 0 \
                            --severity ${params.TRIVY_SEVERITY} \
                            --format table \
                            --output trivy-fs-report.txt \
                            .
                    else
                        echo "trivy CLI not found in PATH, falling back to aquasec/trivy container..."
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v "${WORKSPACE}":/workspace \
                            aquasec/trivy:latest fs \
                            --exit-code 0 \
                            --severity ${params.TRIVY_SEVERITY} \
                            --format table \
                            --output /workspace/trivy-fs-report.txt \
                            /workspace
                    fi
                    cat trivy-fs-report.txt || true
                """
            }
        }

        // ================================================================
        // 5. Docker Build
        // ================================================================
        stage('Docker Build') {
            steps {
                script {
                    def services = getServices()
                    for (service in services) {
                        def imgTag = "${params.DOCKER_HUB_USER}/${service.name}:${params.IMAGE_TAG}"
                        def imgLatest = "${params.DOCKER_HUB_USER}/${service.name}:latest"
                        echo "Building Docker image for ${service.name} -> ${imgTag}..."
                        sh """
                            docker build \
                                -t ${imgTag} \
                                -t ${imgLatest} \
                                ${service.path}
                        """
                    }
                }
            }
        }

        // ================================================================
        // 6. Trivy Container Image Scan
        // ================================================================
        stage('Trivy Container Scan') {
            steps {
                script {
                    def services = getServices()
                    for (service in services) {
                        def imgTag = "${params.DOCKER_HUB_USER}/${service.name}:${params.IMAGE_TAG}"
                        def reportFile = "trivy-image-${service.name}-report.txt"
                        echo "Running Trivy container scan on ${imgTag}..."
                        sh """
                            if command -v trivy >/dev/null 2>&1; then
                                trivy image \
                                    --exit-code 0 \
                                    --severity ${params.TRIVY_SEVERITY} \
                                    --format table \
                                    --output ${reportFile} \
                                    ${imgTag}
                            else
                                echo "trivy CLI not found in PATH, scanning image via aquasec/trivy container..."
                                docker run --rm \
                                    -v /var/run/docker.sock:/var/run/docker.sock \
                                    -v "${WORKSPACE}":/workspace \
                                    aquasec/trivy:latest image \
                                    --exit-code 0 \
                                    --severity ${params.TRIVY_SEVERITY} \
                                    --format table \
                                    --output /workspace/${reportFile} \
                                    ${imgTag}
                            fi
                            head -n 50 ${reportFile} || true
                        """
                    }
                }
            }
        }

        // ================================================================
        // 7. Push Images to Docker Hub
        // ================================================================
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_CREDENTIALS_ID,
                    usernameVariable: 'DOCKER_HUB_USERNAME',
                    passwordVariable: 'DOCKER_HUB_PASSWORD'
                )]) {
                    echo 'Authenticating with Docker Hub...'
                    sh 'echo "$DOCKER_HUB_PASSWORD" | docker login -u "$DOCKER_HUB_USERNAME" --password-stdin'

                    script {
                        def services = getServices()
                        for (service in services) {
                            def imgTag = "${params.DOCKER_HUB_USER}/${service.name}:${params.IMAGE_TAG}"
                            def imgLatest = "${params.DOCKER_HUB_USER}/${service.name}:latest"
                            echo "Pushing ${imgTag} and ${imgLatest} to Docker Hub..."
                            sh "docker push ${imgTag}"
                            sh "docker push ${imgLatest}"
                        }
                    }

                    echo 'Logging out of Docker Hub...'
                    sh 'docker logout'
                }
            }
        }
    }

    post {
        always {
            echo 'Archiving scan reports...'
            archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true

            echo 'Cleaning up local Docker images...'
            script {
                def services = getServices()
                for (service in services) {
                    sh """
                        docker rmi ${params.DOCKER_HUB_USER}/${service.name}:${params.IMAGE_TAG} || true
                        docker rmi ${params.DOCKER_HUB_USER}/${service.name}:latest || true
                    """
                }
            }

            cleanWs()
        }
        success {
            echo "=========================================================="
            echo " Pipeline Succeeded! All images built, scanned, and pushed."
            echo "=========================================================="
        }
        failure {
            echo "=========================================================="
            echo " Pipeline Failed! Please check the stage logs above."
            echo "=========================================================="
        }
    }
}
