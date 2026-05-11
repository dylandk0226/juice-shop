pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 120, unit: 'MINUTES')
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
      options { timeout(time: 90, unit: 'MINUTES') }
      steps {
        sh '''
          # Clean any leftover state from previous runs
          rm -rf ${WORKSPACE}/zap-reports
          mkdir -p ${WORKSPACE}/zap-reports
          docker rm -f zap-scan 2>/dev/null || true
          docker volume rm zap-wrk 2>/dev/null || true

          # Create the named volume and chmod 777 the /zap/wrk directory
          # so the non-root zap user (UID 1000) can write reports into it.
          # We must NOT run ZAP as root (--user 0:0) because Firefox used
          # by the AJAX spider refuses to run as root.
          docker volume create zap-wrk
          docker run --rm --user 0:0 -v zap-wrk:/zap/wrk alpine:latest \
            chmod 777 /zap/wrk

          # Run ZAP using the zap-full-scan.py wrapper script.
          # We use the wrapper instead of the Automation Framework because
          # zap-full-scan.py's -r flag generates ZAP's built-in legacy HTML
          # report which includes the ZAP logo, color-coded risk badges, and
          # full inline CSS styling — the polished report format we want.
          # The Automation Framework's traditional-html-plus template uses
          # the reports add-on which produces a much plainer output without
          # the styling.
          #
          # ZAP_JVM_OPTIONS=-Xmx4g gives ZAP 4 GB heap. The default auto-sized
          # heap (~2 GB) is too small for a full active scan on Juice Shop,
          # causing the JVM to OOM and crash mid-scan before reports can be
          # generated.
          docker run --name zap-scan \
            -e ZAP_JVM_OPTIONS="-Xmx4g" \
            --memory=6g \
            --network dast_juice-staging \
            -v zap-wrk:/zap/wrk \
            zaproxy/zap-stable \
            zap-full-scan.py \
              -t http://juice-shop-staging:3000 \
              -r full-scan-report.html \
              -J full-scan-report.json \
              -w full-scan-report.md \
              -j \
            || true

          # Extract reports into Jenkins workspace via docker cp
          docker cp zap-scan:/zap/wrk/full-scan-report.html ${WORKSPACE}/zap-reports/ || true
          docker cp zap-scan:/zap/wrk/full-scan-report.json ${WORKSPACE}/zap-reports/ || true
          docker cp zap-scan:/zap/wrk/full-scan-report.md  ${WORKSPACE}/zap-reports/ || true

          # Clean up container and volume
          docker rm zap-scan || true
          docker volume rm zap-wrk 2>/dev/null || true

          # Sanity check — list what we have
          echo "=== Final contents of zap-reports/ ==="
          ls -la ${WORKSPACE}/zap-reports/

          # Fail the stage explicitly if the HTML report is missing.
          # This catches silent failures like ZAP OOM-ing mid-scan
          # before reports are generated.
          if [ ! -s "${WORKSPACE}/zap-reports/full-scan-report.html" ]; then
            echo "ERROR: full-scan-report.html is missing or empty."
            echo "Likely cause: ZAP crashed before report job ran."
            echo "Check Java heap settings (ZAP_JVM_OPTIONS) and stage 7 console for OOM."
            exit 1
          fi
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
      sh '''
        docker rm -f juice-shop-staging 2>/dev/null || true
        docker rm -f zap-scan 2>/dev/null || true
        docker volume rm zap-wrk 2>/dev/null || true
      '''
      echo "Build #${env.BUILD_NUMBER} finished with status: ${currentBuild.currentResult}"
    }
  }
}
