pipeline {
    agent any

    environment {
        STAGING_SERVER = 'user@vue-nginx-1'
        REMOTE_PATH = '/var/www/html'
        STAGING_URL = "http://vue-nginx-1:80 "
        SEMGREP_BIN = "/opt/jenkins-venv/bin/semgrep"
        TRIVY_BIN = "/usr/local/bin/trivy"
        ZAP_BIN = "/opt/zaproxy/zap.sh"
        DEP_CHECK_BIN = "/opt/dependency-check/bin/dependency-check.sh"
        NPM = "/home/user/.nvm/versions/node/v18.20.8/bin/npm"
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

        // stage('SCA - Dependency Check') {
        //     steps {
        //         echo "Running OWASP Dependency-Check..."
        //         sh """
        //             mkdir -p dependency-check-reports
        //              ${DEP_CHECK_BIN} --noupdate --project "devsecops-labs" --scan . --format JSON --out dependency-check-reports || true
        //         """
        //         archiveArtifacts artifacts: 'dependency-check-reports/**', allowEmptyArchive: true
        //     }
        // }

        // stage('Build') {
        //     steps {
        //         echo "Building app (npm install and tests)..."
        //         sh '''
        //             cd src
        //             npm install --no-audit --no-fund
        //             if [ -f package.json ]; then
        //                 if npm test --silent; then
        //                     echo "Tests OK"
        //                 else
        //                     echo "Tests failed (continue)"
        //                 fi
        //             fi
        //         '''
        //     }
        // }

        stage('Deploy to Staging') {
            steps {
                echo "Deploying to staging with docker-compose..."
                sshagent(['vue-nginx-1']) {
                    sh "scp -o StrictHostKeyChecking=no -r src/* ${STAGING_SERVER}:${REMOTE_PATH}/"
                    sh "ssh -o StrictHostKeyChecking=no user@vue-nginx-1 bash -c export PATH=/home/user/.nvm/versions/node/v18.20.8/bin:$PATH cd /var/www/html npm install --no-audit --no-fund npm run start"
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
