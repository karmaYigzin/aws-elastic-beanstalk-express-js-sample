pipeline {
  agent none

  options {
    timeout(time: 30, unit: 'MINUTES')
    timestamps()
    skipDefaultCheckout()
    buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '20', artifactDaysToKeepStr: '14', artifactNumToKeepStr: '10'))
  }

  environment {
    DOCKER_HOST       = 'tcp://docker:2376'
    DOCKER_TLS_VERIFY = '1'
    DOCKER_CERT_PATH  = '/certs/client'
    IMAGE_REPO        = 'yigz/karma_22026392' 
    IMAGE_TAG         = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      agent any
      steps {
        checkout scm
        script {
          env.GIT_COMMIT        = sh(script: 'git rev-parse HEAD',              returnStdout: true).trim()
          env.GIT_COMMIT_SHORT  = sh(script: 'git rev-parse --short=7 HEAD',    returnStdout: true).trim()
          env.BRANCH_NAME       = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
          env.IMAGE_TAG         = env.GIT_COMMIT_SHORT
        } 
          echo'Checkout completed'
      }
    }

    stage('Install & Test (Node16)') {
      agent { docker { image 'node:16' } }
      steps {
        sh 'npm ci || npm install'
        sh 'npm test || true'
        echo "tests completed"
      }
    }

    stage('Dependency Scan (Snyk)') {
      agent { docker { image 'node:16' } }
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            npm i -g snyk
            snyk config set api="$SNYK_TOKEN"
            snyk test --severity-threshold=high --json > snyk-deps.json  
            SNYK_EXIT_CODE=$?
            if [ $SNYK_EXIT_CODE -ne 0 ]; then
              echo "High vulnerabilities found" 
            else
              echo "High vulnerabilities not found."
            fi
          '''
          archiveArtifacts artifacts: 'snyk-deps.json', allowEmptyArchive: true, fingerprint: true
          echo "Dependency scan completed."
        }
      }
    }

    stage('Build Image') {
      agent any
      steps {
        sh '''
          docker version
          docker build -t ${IMAGE_REPO}:${IMAGE_TAG} .
          docker tag ${IMAGE_REPO}:${IMAGE_TAG} ${IMAGE_REPO}:latest
        '''
      }
    }

    stage('Push Image') {
      agent any
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE_REPO}:${IMAGE_TAG}
            docker push ${IMAGE_REPO}:latest
          '''
        }
      }
    }

    stage('Container Scan (Snyk)') {
      when { expression { fileExists('Dockerfile') } }
      agent any
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            docker run --rm \
              -e SNYK_TOKEN="$SNYK_TOKEN" \
              -v /var/run/docker.sock:/var/run/docker.sock \
              snyk/snyk:docker container test ${IMAGE_REPO}:${IMAGE_TAG} \
              --severity-threshold=high --json > snyk-container.json || exit 1
          '''
          archiveArtifacts artifacts: 'snyk-container.json', allowEmptyArchive: true, fingerprint: true
        }
      }
    }
  }

  post {
    always {
      node('built-in') {
        sh 'docker logout || true'
        cleanWs(deleteDirs: true, notFailBuild: true)
        archiveArtifacts artifacts: '**/snyk*.log, **/snyk*.json,**/test-results.log, **/build.log', allowEmptyArchive: true, fingerprint: true
      }
    }
    success { echo "Pushed ${IMAGE_REPO}:${IMAGE_TAG}" }
    failure { echo 'Pipeline failed' }
  }
}
