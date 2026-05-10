pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
  }

  tools {
    nodejs 'NodeJS-20'
  }

  environment {
    SONAR_PROJECT_KEY = 'juice-shop'
  }

  stages {

    stage('1. Checkout') {
      steps {
        checkout scm
      }
    }

    stage('2. Install dependencies') {
      steps {
        sh 'node --version || true'
        sh 'npm --version || true'
        sh 'npm install --no-audit --no-fund --ignore-scripts'
      }
    }

    stage('3. SAST — SonarQube') {
      steps {
        script {
          def scannerHome = tool 'SonarScanner'
          withSonarQubeEnv('SonarQube') {
            sh "${scannerHome}/bin/sonar-scanner"
          }
        }
      }
    }

    stage('4. Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: false
        }
      }
    }

    stage('5. SCA')             { steps { echo 'TODO Member 2 — Dependency-Check + Retire.js' } }
    stage('6. Build & Deploy') {
  steps {
    sh '''
      # Build the image from the source we just checked out
      docker build -t juice-shop:${BUILD_NUMBER} .

      # Bring up staging on the dast_juice-staging network
      docker network create dast_juice-staging || true
      docker rm -f juice-shop-staging || true
      docker run -d \
        --name juice-shop-staging \
        --network dast_juice-staging \
        -p 3000:3000 \
        -e NODE_ENV=unsafe \
        juice-shop:${BUILD_NUMBER}

      # Wait until it's responding
      for i in $(seq 1 30); do
        if curl -sf http://localhost:3000 >/dev/null; then
          echo "Staging is up"; exit 0
        fi
        sleep 2
      done
      echo "Staging never came up"; exit 1
    '''
  }
}
    stage('7. DAST — OWASP ZAP') {
  steps {
    sh '''
      mkdir -p ${WORKSPACE}/zap-reports
      chmod 777 ${WORKSPACE}/zap-reports   # ZAP container writes as non-root

      docker run --rm \
        --memory=6g \
        --network dast_juice-staging \
        -v ${WORKSPACE}/zap-reports:/zap/wrk/:rw \
        zaproxy/zap-stable \
        zap-full-scan.py \
          -t http://juice-shop-staging:3000 \
          -r full-scan-report.html \
          -J full-scan-report.json \
          -w full-scan-report.md \
          -z "-Xmx4g" \
        || true   # ZAP exits non-zero on findings; do not fail build (yet)
    '''
  }
}
    stage('8. Publish reports') {
  steps {
    archiveArtifacts artifacts: 'zap-reports/*', allowEmptyArchive: true
    publishHTML(target: [
      allowMissing: false,
      alwaysLinkToLastBuild: true,
      keepAll: true,
      reportDir: 'zap-reports',
      reportFiles: 'full-scan-report.html',
      reportName: 'OWASP ZAP DAST Report'
    ])
  }
}
  }

  post {
  always {
    sh 'docker rm -f juice-shop-staging || true'
    echo "Build #${env.BUILD_NUMBER} finished with status: ${currentBuild.currentResult}"
  }
}
