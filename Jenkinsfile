pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'ikomajic'
        IMAGE_NAME = 'project2-demo'
        IMAGE_TAG = "build-%BUILD_NUMBER%"
        KUBECONFIG = 'C:\\Users\\ivanm\\.kube\\config'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t %DOCKERHUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% ."
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    bat "echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                bat "docker push %DOCKERHUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            environment {
                KUBECONFIG = 'C:\\Users\\ivanm\\.kube\\config'
            }    
            steps {
                bat """
                helm upgrade --install project2 helm ^
                  --set image.repository=%DOCKERHUB_USER%/%IMAGE_NAME% ^
                  --set image.tag=%IMAGE_TAG%
                """
            }
        }
    }
}
