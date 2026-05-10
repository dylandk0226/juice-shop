pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 90, unit: 'MINUTES')
  }

  tools {
    nodejs 'NodeJS-20'
  }

  environment {
    SONAR_PROJECT_KEY = 'juice-shop'
  }

  parameters {
    choice(
      name: 'RUN_MODE',
      choices: ['dast-only', 'all'],
      description: 'dast-only = skip SAST + Quality Gate (Member 1 — DAST work). all = full pipeline (use once teammates merge their stages).'
    )
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
      when { expression { params.RUN_MODE == 'all' } }
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
      when { expression { params.RUN_MODE == 'all' } }
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

      # Wait until it's responding — health-check from inside the
      # dast_juice-staging network (Jenkins itself isn't on it, and
      # 'localhost' inside the Jenkins container != the staging host).
      for i in $(seq 1 60); do
        if docker run --rm --network dast_juice-staging curlimages/curl:8.10.1 \
             -sf http://juice-shop-staging:3000 >/dev/null 2>&1; then
          echo "Staging is up after ${i} attempts (~$((i*3))s)"
          exit 0
        fi
        sleep 3
      done
      echo "Staging never came up — last 50 lines of container logs:"
      docker logs --tail 50 juice-shop-staging || true
      exit 1
    '''
  }
}
    stage('7. DAST — OWASP ZAP') {
      options { timeout(time: 60, unit: 'MINUTES') }
      steps {
        sh '''
          # Clean any leftover files from previous runs.
          # Files written by ZAP (non-root user) may not be removable by Jenkins,
          # so wipe via a privileged throwaway container.
          docker run --rm -v ${WORKSPACE}/zap-reports:/clean alpine:latest \
            sh -c "rm -rf /clean/* /clean/.[!.]* 2>/dev/null || true" || true

          mkdir -p ${WORKSPACE}/zap-reports
          chmod 777 ${WORKSPACE}/zap-reports

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
            || true

          # Sanity check that reports were actually written
          ls -la ${WORKSPACE}/zap-reports/ || true
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
}
