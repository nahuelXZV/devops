pipeline {
    agent any

    environment {
        GIT_CREDENTIALS = "git-credentials"
        DOCKER_IMAGE_NAME = "devsecops-labs/app:latest"
        SSH_CREDENTIALS = "ssh-deploy-key"
        STAGING_URL = "http://localhost:3000"
        SEMGREP_BIN = "/opt/jenkins-venv/bin/semgrep"
        TRIVY_BIN = "/usr/local/bin/trivy"
        ZAP_BIN = "/opt/zaproxy/zap.sh"
        DEP_CHECK_HOME = "/home/jenkins/dependency-check"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }

        stage('SAST - Semgrep') {
            steps {
                echo "Running Semgrep (SAST)..."
                sh """
                    ${SEMGREP_BIN} --config=auto --json --output semgrep-results.json src || true
                    cat semgrep-results.json || true
                """
                archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
            }
            post {
                always {
                    echo "Semgrep scan finished."
                }
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                echo "Running OWASP Dependency-Check..."
                sh """
                    mkdir -p dependency-check-reports
                    ${DEP_CHECK_HOME}/bin/dependency-check.sh --project "devsecops-labs" --scan . --format JSON --out dependency-check-reports || true
                """
                archiveArtifacts artifacts: 'dependency-check-reports/**', allowEmptyArchive: true
            }
        }

        stage('Build') {
            steps {
                echo "Building app (npm install and tests)..."
                sh '''
                    cd src
                    npm install --no-audit --no-fund
                    if [ -f package.json ]; then
                        if npm test --silent; then
                            echo "Tests OK"
                        else
                            echo "Tests failed (continue)"
                        fi
                    fi
                '''
            }
        }

        stage('Docker Build & Trivy Scan') {
            steps {
                echo "Building Docker image..."
                sh """
                    docker build -t ${DOCKER_IMAGE_NAME} -f Dockerfile .
                """
                echo "Scanning image with Trivy..."
                sh """
                    mkdir -p trivy-reports
                    ${TRIVY_BIN} image --format json --output trivy-reports/trivy-report.json ${DOCKER_IMAGE_NAME} || true
                    ${TRIVY_BIN} image --severity HIGH,CRITICAL ${DOCKER_IMAGE_NAME} || true
                """
                archiveArtifacts artifacts: 'trivy-reports/**', allowEmptyArchive: true
            }
        }

        stage('Deploy to Staging (docker-compose)') {
            steps {
                echo "Deploying to staging with docker-compose..."
                sh '''
                    docker-compose -f docker-compose.yml down || true
                    docker-compose -f docker-compose.yml up -d --build
                    sleep 8
                    docker ps -a
                '''
            }
        }

        stage('DAST - OWASP ZAP scan') {
            steps {
                echo "Running DAST (OWASP ZAP) against ${STAGING_URL}..."
                sh """
                    mkdir -p zap-reports
                    ${ZAP_BIN} -daemon -host 0.0.0.0 -port 8080 -config api.disablekey=true
                    sleep 10
                    curl -s ${STAGING_URL} || true
                    ${ZAP_BIN} -cmd -quickurl ${STAGING_URL} -quickout zap-reports/zap-report.html || true
                """
                archiveArtifacts artifacts: 'zap-reports/**', allowEmptyArchive: true
            }
        }

        stage('Policy Check - Fail on HIGH/CRITICAL CVEs') {
            steps {
                sh '''
                    chmod +x scripts/scan_trivy_fail.sh
                    ./scripts/scan_trivy_fail.sh $DOCKER_IMAGE_NAME || exit_code=$?
                    if [ "${exit_code:-0}" -eq 2 ]; then
                        echo "Failing pipeline due to HIGH/CRITICAL vulnerabilities detected by Trivy."
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Collecting artifacts..."
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
