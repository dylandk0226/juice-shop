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
        sh 'npm ci --no-audit --no-fund'
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
    stage('6. Build & Deploy')  { steps { echo 'TODO Member 1 — docker build + run' } }
    stage('7. DAST')            { steps { echo 'TODO Member 1 — OWASP ZAP baseline' } }
    stage('8. Publish reports') { steps { echo 'TODO — archive HTML reports' } }
  }

  post {
    always  { echo "Build #${env.BUILD_NUMBER} finished with status: ${currentBuild.currentResult}" }
    failure { echo 'Pipeline failed — check the failed stage logs.' }
  }
}
