pipeline {
  agent any

  environment {
    GIT_CREDENTIALS = "git-credentials"
    DOCKER_IMAGE_NAME = "devsecops-labs/app:latest"
    SSH_CREDENTIALS = "ssh-deploy-key"
    STAGING_URL = "http://localhost:3000"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('SAST - Semgrep') {
      agent {
        docker { image 'returntocorp/semgrep:latest' }
      }
      steps {
        echo "Running Semgrep (SAST)..."
        sh '''
          semgrep --config=auto --json --output semgrep-results.json src || true
          cat semgrep-results.json || true
        '''
        archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
      }
      post {
        always {
          script { sh 'echo "Semgrep done."' }
        }
      }
    }

    stage('SCA - Dependency Check (OWASP dependency-check)') {
      agent {
        docker { image 'owasp/dependency-check:latest' }
      }
      steps {
        echo "Running SCA / Dependency-Check..."
        sh '''
          mkdir -p dependency-check-reports
          dependency-check --project "devsecops-labs" --scan . --format JSON --out dependency-check-reports || true
        '''
        archiveArtifacts artifacts: 'dependency-check-reports/**', allowEmptyArchive: true
      }
    }

    stage('Build') {
      agent { label 'docker' }
      steps {
        echo "Building app (npm install and tests)..."
        sh '''
          cd src
          npm install --no-audit --no-fund
          if [ -f package.json ]; then
            if npm test --silent; then echo "Tests OK"; else echo "Tests failed (continue)"; fi
          fi
        '''
      }
    }

    stage('Docker Build & Trivy Scan') {
      agent { label 'docker' }
      steps {
        echo "Building Docker image..."
        sh '''
          docker build -t ${DOCKER_IMAGE_NAME} -f Dockerfile .
        '''
        echo "Scanning image with Trivy..."
        sh '''
          mkdir -p trivy-reports
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --format json --output trivy-reports/trivy-report.json ${DOCKER_IMAGE_NAME} || true
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL ${DOCKER_IMAGE_NAME} || true
        '''
        archiveArtifacts artifacts: 'trivy-reports/**', allowEmptyArchive: true
      }
    }
    stage('Deploy to Staging (docker-compose)') {
      agent { label 'docker' }
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
      agent { label 'docker' }
      steps {
        echo "Running DAST (OWASP ZAP) against ${STAGING_URL} ..."
        sh '''
          mkdir -p zap-reports
          docker run --rm --network host owasp/zap2docker-stable zap-baseline.py -t ${STAGING_URL} -r zap-reports/zap-report.html || true
        '''
        archiveArtifacts artifacts: 'zap-reports/**', allowEmptyArchive: true
      }
    }

  } // stages

  post {
    always {
      echo "Pipeline finished. Collecting artifacts..."
    }
    failure {
      echo "Pipeline failed!"
    }
  }
}
