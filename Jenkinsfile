pipeline {
    agent any

    environment {
        GIT_CREDENTIALS = "git-credentials"
        DOCKER_IMAGE_NAME = "devsecops-labs/app:latest"
        SSH_CREDENTIALS = "ssh-deploy-key"
        STAGING_URL = "http://vue-nginx-1:80 "
        SEMGREP_BIN = "/opt/jenkins-venv/bin/semgrep"
        TRIVY_BIN = "/usr/local/bin/trivy"
        ZAP_BIN = "/opt/zaproxy/zap.sh"
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
                    dependency-check.sh --project "devsecops-labs" --scan . --format JSON --out dependency-check-reports || true
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
                    npm run build
                '''
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo "Deploying to staging with docker-compose..."
                sshagent(['vue-nginx-1']) {
                    sh "scp -o StrictHostKeyChecking=no -r dist/* ${STAGING_SERVER}:${REMOTE_PATH}/"
                }
            }
        }

        stage('DAST - OWASP ZAP scan') {
            steps {
                echo "Running DAST (OWASP ZAP) against ${STAGING_URL}..."
                sh """
                    mkdir -p zap-reports
                    ${ZAP_BIN} -daemon -host 0.0.0.0 -port 3000 -config api.disablekey=true
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
