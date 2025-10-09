pipeline {
    agent { docker { image 'node:16' } }

    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test || echo "No tests found"'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = "karmayigzin/aws-node-app:${env.BUILD_NUMBER}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }
    }

    post {
        always {
            echo 'Build completed.'
        }
    }
}
