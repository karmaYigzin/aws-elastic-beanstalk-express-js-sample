pipeline {
    agent none

    environment {
        DOCKER_HOST = 'tcp://docker:2376'
        DOKCER_TLS_VERIFY = '1'
        DOCKER_CERT_PATH = '/certs/client'
        IMAGE_REPO = "https://github.com/karmaYigzin/aws-elastic-beanstalk-express-js-sample.git"  
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            agent any
            steps { 
                checkout scm 
                script {
                    env.GIT_COMMIT = sh(script : 'git rev-parse HEAD', returnStdout: true).trim()
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
                    env.BRANCH_NAME = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = env.GIT_COMMIT_SHORT
                    echo "Branch: ${env.branch_name}"
                    echo"Checkout Stage succesful"
                }
            }
        }
        stage('Install & Test(Node)') {
            agent { docker { image 'node:16' } }  
            steps {
                sh 'npm ci || npm install'
                sh 'npm test || echo "No tests found"'
            }
        }
     

        stage('Security Scan (Snyk)') {
            agent { docker { image 'node:16' } }
            steps {
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh 'npm i -g snyk'
                    sh 'snyk auth "$SNYK_TOKEN"'
                    scrpit {
                        def code = sh(script: 'snyk test --severity-threshold=high --json > snyk-deps.json', returnStatus: true)
                        archiveArtifacts artifacts: 'synk-deps.json', allowEmptyArchive: true, fingerprint: true
                        if (code != 0) {
                           error 'High/Critical dependency vulnerabilities found.'
                        }
                    }
                }
            }
        }
         stage('Build Docker Image') {
            agent any
            steps {
                sh '''
                  which docker
                  docker version
                  docker build -t ${IMAGE_REPO}:${IMAGE_TAG} .
                  docker tag ${IMAGE_REPO}:${IMAGE_TAG} ${IMAGE_REPO}:latest
                '''
            }
        }

        stage('Push to Docker Hub') {
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
    
        stage(' Container Scan (Synk)' ) {
            when { expression { fileExistis('Dockerfile')}}
            agent any
            steps {
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh '''
'                       docker run --rm \
                            -e SYNK_TOKEN="$SYNK_TOKEN" \
                            -V /var/run/docker.sock:/var/run/docker.sock \
                            snyk/snyk:docker container test ${IMAGE_REPO}:${IMAGE_TAG} --severity-threshold=high --json > snyk-container.json || exit 1
                    ''''
                    archiveArtifacts artifacts: 'synk-container.json', allowEmptyArchive: true, fingerprint: true
                }
            }
        }
    }

    post {
       always {
           sh 'docker logout || true'
           cleanWs(deleteDirs: true, notFailBuild: true)
       }
       success { echo "Pushed ${IMAGE_REPO}:${IMAGE_TAG}"}
       FAILURE { echo "Pipeline failed"}
    }
}

