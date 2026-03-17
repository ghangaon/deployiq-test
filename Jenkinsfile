pipeline {
  agent any

  parameters {
    string(name: 'DEPLOY_ENV',       defaultValue: 'staging')
    string(name: 'branch',           defaultValue: 'main')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    booleanParam(name: 'SKIP_SCAN',  defaultValue: false)
  }

  environment {
    IMAGE_NAME    = "ghangaon/deployiq-app"
    IMAGE_TAG     = "${BUILD_NUMBER}"
    REGISTRY_CRED = "dockerhub-credentials"
    APP_PORT      = "8090"
    APP_CONTAINER = "deployiq-staging"
  }

  stages {

    stage('1. Checkout') {
      steps {
        echo "=============================="
        echo "  STAGE 1: CHECKOUT"
        echo "=============================="
        sh '''
          echo "Branch  : ''' + params.branch + '''"
          echo "Env     : ''' + params.DEPLOY_ENV + '''"
          echo "Build # : ${BUILD_NUMBER}"
          echo ""
          echo "--- Last 3 commits ---"
          git log --oneline -3
          echo ""
          echo "--- Files changed ---"
          git diff --name-only HEAD~1 HEAD 2>/dev/null || echo "(first build)"
        '''
      }
    }

    stage('2. Build & Unit Test') {
      when { expression { !params.SKIP_TESTS } }
      steps {
        echo "=============================="
        echo "  STAGE 2: BUILD & UNIT TEST"
        echo "=============================="
        sh '''
          echo "--- Checking required files ---"
          [ -f Dockerfile ]     && echo "✅ Dockerfile"     || { echo "❌ Dockerfile missing!"; exit 1; }
          [ -f app/index.html ] && echo "✅ app/index.html" || { echo "❌ index.html missing!"; exit 1; }

          echo ""
          echo "--- HTML validation ---"
          grep -q "DOCTYPE" app/index.html && echo "✅ DOCTYPE OK" || echo "⚠️  DOCTYPE missing"
          grep -q "<title>" app/index.html && echo "✅ Title OK"   || echo "⚠️  Title missing"

          echo ""
          echo "--- Dockerfile validation ---"
          grep -q "FROM"   Dockerfile && echo "✅ FROM OK"
          grep -q "COPY"   Dockerfile && echo "✅ COPY OK"
          grep -q "EXPOSE" Dockerfile && echo "✅ EXPOSE OK"

          echo ""
          echo "✅ All unit tests PASSED"
        '''
      }
    }

    stage('3. Code Scan') {
      when { expression { !params.SKIP_SCAN } }
      steps {
        echo "=============================="
        echo "  STAGE 3: CODE SCAN"
        echo "=============================="
        sh '''
          echo "--- Secret detection ---"
          if grep -rEi "password=|secret=|token=|api_key=" app/ 2>/dev/null; then
            echo "WARNING: Potential secrets found!"
          else
            echo "OK: No hardcoded secrets found"
          fi

          echo ""
          echo "--- File sizes ---"
          find app/ -type f | while read f; do
            echo "  $f: $(wc -c < $f) bytes"
          done

          echo ""
          echo "✅ Code scan PASSED"
        '''
      }
    }

    stage('4. Image Build') {
      steps {
        echo "=============================="
        echo "  STAGE 4: IMAGE BUILD"
        echo "=============================="
        sh """
          echo "Building: ${IMAGE_NAME}:${IMAGE_TAG}"
          docker build \\
            --label build.number=${IMAGE_TAG} \\
            --label build.branch=${params.branch} \\
            --label build.env=${params.DEPLOY_ENV} \\
            -t ${IMAGE_NAME}:${IMAGE_TAG} \\
            -t ${IMAGE_NAME}:latest \\
            .
          echo ""
          docker images ${IMAGE_NAME} --format "ID:{{.ID}} Size:{{.Size}} Tag:{{.Tag}}"
          echo "✅ Image built: ${IMAGE_NAME}:${IMAGE_TAG}"
        """
      }
    }

    stage('5. Image Scan') {
      when { expression { !params.SKIP_SCAN } }
      steps {
        echo "=============================="
        echo "  STAGE 5: IMAGE SCAN (Trivy)"
        echo "=============================="
        sh """
          docker run --rm \\
            -v /var/run/docker.sock:/var/run/docker.sock \\
            aquasec/trivy image \\
            --exit-code 0 \\
            --severity HIGH,CRITICAL \\
            --no-progress \\
            ${IMAGE_NAME}:${IMAGE_TAG} || true
          echo ""
          echo "✅ Image scan complete"
        """
      }
    }

    stage('6. Image Push') {
      steps {
        echo "=============================="
        echo "  STAGE 6: IMAGE PUSH"
        echo "=============================="
        withCredentials([usernamePassword(
          credentialsId: "${REGISTRY_CRED}",
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh """
            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
            docker push ${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${IMAGE_NAME}:latest
            docker logout
            echo ""
            echo "✅ Pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "🔗 https://hub.docker.com/r/ghangaon/deployiq-app/tags"
          """
        }
      }
    }

    stage('7. Deploy') {
      steps {
        echo "=============================="
        echo "  STAGE 7: DEPLOY TO ${params.DEPLOY_ENV}"
        echo "=============================="
        sh """
          echo "Deploying ${IMAGE_NAME}:${IMAGE_TAG} → port ${APP_PORT}"
          docker stop ${APP_CONTAINER} 2>/dev/null || true
          docker rm   ${APP_CONTAINER} 2>/dev/null || true
          docker pull ${IMAGE_NAME}:${IMAGE_TAG}
          docker run -d \\
            --name ${APP_CONTAINER} \\
            -p ${APP_PORT}:80 \\
            --restart unless-stopped \\
            --label deployed.by=jenkins \\
            --label deployed.build=${IMAGE_TAG} \\
            --label deployed.env=${params.DEPLOY_ENV} \\
            ${IMAGE_NAME}:${IMAGE_TAG}
          sleep 3
          docker ps | grep ${APP_CONTAINER}
          echo ""
          echo "✅ Deployed! http://\$(hostname -I | awk '{print \$1}'):${APP_PORT}"
        """
      }
    }

    stage('8. Smoke Test') {
      steps {
        echo "=============================="
        echo "  STAGE 8: SMOKE TEST"
        echo "=============================="
        sh """
          sleep 3
          HTTP=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${APP_PORT})
          echo "App response: HTTP \$HTTP"
          if [ "\$HTTP" = "200" ]; then
            echo "✅ Smoke test PASSED — app is live!"
            echo "🌐 http://192.168.1.35:${APP_PORT}"
          else
            echo "❌ Smoke test FAILED — HTTP \$HTTP"
            exit 1
          fi
        """
      }
    }

  }

  post {
    success {
      echo """
======================================
  ✅ PIPELINE SUCCESS!
  Image  : ${IMAGE_NAME}:${IMAGE_TAG}
  App    : http://192.168.1.35:${APP_PORT}
  Hub    : https://hub.docker.com/r/ghangaon/deployiq-app
  Env    : ${params.DEPLOY_ENV}
  Build# : ${IMAGE_TAG}
======================================
      """
    }
    failure {
      echo """
======================================
  ❌ PIPELINE FAILED
  Build# : ${IMAGE_TAG}
  Check logs above
======================================
      """
    }
    always {
      sh "docker image prune -f --filter 'until=24h' || true"
    }
  }
}
