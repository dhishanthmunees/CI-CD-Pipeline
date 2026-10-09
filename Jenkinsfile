pipeline {
    agent any

    environment {
        DOCKERHUB_CREDS = credentials('docker-creds')
        IMAGE_NAME      = "dhish01/devops-webserver"
        APP_SERVER      = "ubuntu@3.107.16.180"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER -t $IMAGE_NAME:latest .'
            }
        }

        stage('Test Image') {
            steps {
                sh '''
                docker run -d --name test-$BUILD_NUMBER -p 8081:80 $IMAGE_NAME:$BUILD_NUMBER
                sleep 5
                curl -f http://localhost:8081 || (docker rm -f test-$BUILD_NUMBER; exit 1)
                docker rm -f test-$BUILD_NUMBER
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh '''
                echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                docker push $IMAGE_NAME:$BUILD_NUMBER
                docker push $IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                sshagent(['web-server-ssh']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $APP_SERVER '
                      docker pull $IMAGE_NAME:$BUILD_NUMBER &&
                      docker stop webserver || true &&
                      docker rm webserver || true &&
                      docker run -d --name webserver -p 80:80 --restart always $IMAGE_NAME:$BUILD_NUMBER
                    '
                    """
                }
            }
        }
    }

    post {
        always { sh 'docker logout' }
        success { echo 'Deployment successful!' }
        failure { echo 'Pipeline failed. Check the logs.' }
    }
}
