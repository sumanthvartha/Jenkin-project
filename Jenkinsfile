// ─────────────────────────────────────────────────────────────────────────
// Jenkins Declarative Pipeline — Node.js API → AWS ECR + ECS
// Author: Sumanth | Project: nodejs-jenkins-cicd
// ─────────────────────────────────────────────────────────────────────────

pipeline {

  // Run all stages on the dedicated Linux agent (never on controller)
  agent { label 'linux-agent' }

  // ── Environment variables available to all stages ──────────────────────
  environment {
    APP_NAME    = 'nodejs-api'
    AWS_REGION  = 'ap-south-1'
    ECR_REGISTRY= '123456789.dkr.ecr.ap-south-1.amazonaws.com'
    IMAGE_TAG   = "${env.BUILD_NUMBER}"       // unique tag per build
    IMAGE_NAME  = "${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"

    // Bind AWS credentials from Jenkins Credentials Manager
    AWS_ACCESS_KEY_ID     = credentials('aws-ecr-creds')
    AWS_SECRET_ACCESS_KEY = credentials('aws-ecr-creds')
  }

  // ── Triggers ────────────────────────────────────────────────────────────
  triggers {
    githubPush()   // fires on GitHub webhook push event
  }

  // ── Pipeline options ─────────────────────────────────────────────────────
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))  // keep last 10 builds
    timeout(time: 30, unit: 'MINUTES')              // kill hung builds
    timestamps()                                       // timestamp every log line
    disableConcurrentBuilds()                          // no parallel runs on same branch
  }

  // ══════════════════════════════════════════════════════════════════════════
  //   STAGES
  // ══════════════════════════════════════════════════════════════════════════
  stages {

    // ── 1. CHECKOUT ───────────────────────────────────────────────────────
    stage('Checkout') {
      steps {
        checkout scm       // clone repo using job's SCM config
        sh 'echo "Branch: ${GIT_BRANCH} | Commit: ${GIT_COMMIT}"'
      }
    }

    // ── 2. INSTALL DEPENDENCIES ──────────────────────────────────────────
    stage('Install') {
      steps {
        sh 'node --version && npm --version'
        sh 'npm ci'         // ci = clean install, uses package-lock.json exactly
      }
    }

    // ── 3. PARALLEL: LINT + TEST ──────────────────────────────────────────
    stage('Quality checks') {
      parallel {

        stage('Lint') {
          steps {
            sh 'npm run lint'
          }
        }

        stage('Unit tests') {
          steps {
            sh 'npm test -- --reporters=default --reporters=jest-junit'
          }
          post {
            always {
              // Publish JUnit XML results to Jenkins (visible in test report UI)
              junit '**/junit.xml'
            }
          }
        }

      } // end parallel
    }

    // ── 4. DOCKER BUILD ───────────────────────────────────────────────────
    stage('Docker build') {
      steps {
        sh """
          docker build \
            --build-arg BUILD_NUMBER=${BUILD_NUMBER} \
            --build-arg GIT_COMMIT=${GIT_COMMIT} \
            -t ${IMAGE_NAME} \
            -t ${ECR_REGISTRY}/${APP_NAME}:latest .
        """
      }
    }

    // ── 5. SECURITY SCAN ─────────────────────────────────────────────────
    stage('Security scan') {
      steps {
        sh """
          trivy image \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            --no-progress \
            ${IMAGE_NAME}
        """
      }
      post {
        always {
          sh "trivy image --format json -o trivy-report.json ${IMAGE_NAME} || true"
          archiveArtifacts artifacts: 'trivy-report.json'
        }
      }
    }

    // ── 6. PUSH TO ECR ───────────────────────────────────────────────────
    stage('Push to ECR') {
      steps {
        sh """
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${ECR_REGISTRY}

          docker push ${IMAGE_NAME}
          docker push ${ECR_REGISTRY}/${APP_NAME}:latest
        """
      }
    }

    // ── 7. DEPLOY TO STAGING ─────────────────────────────────────────────
    stage('Deploy → Staging') {
      steps {
        sh """
          aws ecs update-service \
            --cluster nodejs-cluster-staging \
            --service ${APP_NAME}-staging \
            --force-new-deployment \
            --region ${AWS_REGION}

          aws ecs wait services-stable \
            --cluster nodejs-cluster-staging \
            --services ${APP_NAME}-staging \
            --region ${AWS_REGION}
        """
        echo "Staging deploy complete. Verify at https://staging.myapp.com/health"
      }
    }

    // ── 8. MANUAL APPROVAL GATE ──────────────────────────────────────────
    stage('Approval gate') {
      when {
        branch 'main'      // only ask approval on main branch builds
      }
      steps {
        timeout(time: 24, unit: 'HOURS') {
          input message: "Deploy build #${BUILD_NUMBER} to PRODUCTION?",
                ok: 'Yes, deploy now'
        }
      }
    }

    // ── 9. DEPLOY TO PRODUCTION ──────────────────────────────────────────
    stage('Deploy → Production') {
      when {
        branch 'main'
      }
      steps {
        sh """
          aws ecs update-service \
            --cluster nodejs-cluster-prod \
            --service ${APP_NAME}-prod \
            --force-new-deployment \
            --region ${AWS_REGION}

          aws ecs wait services-stable \
            --cluster nodejs-cluster-prod \
            --services ${APP_NAME}-prod \
            --region ${AWS_REGION}
        """
      }
    }

  } // end stages

  // ══════════════════════════════════════════════════════════════════════════
  //   POST — runs after all stages regardless of outcome
  // ══════════════════════════════════════════════════════════════════════════
  post {

    success {
      slackSend(
        channel: '#deployments',
        color:   'good',
        message: "✅ SUCCESS — ${APP_NAME} build #${BUILD_NUMBER} deployed! ${BUILD_URL}"
      )
    }

    failure {
      slackSend(
        channel: '#deployments',
        color:   'danger',
        message: "❌ FAILED — ${APP_NAME} build #${BUILD_NUMBER} failed. ${BUILD_URL}"
      )
    }

    always {
      cleanWs()       // delete workspace to free disk space after every run
    }

  }

}