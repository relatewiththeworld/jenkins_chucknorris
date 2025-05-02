pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "relatewiththeworld/chucknorrisapp"
        DOCKERFILE = "Dockerfile"
        DOCKER_REGISTRY = "index.docker.io/v1"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = "dockerhub-creds"
        EC2_USER = "ubuntu"
        EC2_HOST = ""
        SSH_CREDENTIALS_ID = "ec2-ssh-key"
        CONTAINER_NAME = "chucknorris-app"
        HOST_PORT = 80
        CONTAINER_PORT = 3000
    }
    stages {
        stage('Git checkout') {
            steps {
                git url: 'https://github.com/relatewiththeworld/jenkins_chucknorris.git', branch: 'main'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Test') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker Image'
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}", "-f ${DOCKERFILE} .")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                echo 'pushing Docker Image to DockerHub'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                    docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                }
            } 
        }

        stage('Deploy')
    }
}