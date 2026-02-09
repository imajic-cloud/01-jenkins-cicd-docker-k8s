pipeline {
    agent { label 'linux' }

    environment {
        DOCKERHUB_USER = 'ikomajic'
        IMAGE_NAME     = 'project2-demo'
        IMAGE_TAG      = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                  docker build \
                    -t ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      echo "${DOCKER_PASS}" | docker login \
                        -u "${DOCKER_USER}" --password-stdin
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                  docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                sh """
                  helm upgrade --install project2 helm \
                    --set image.repository=${DOCKERHUB_USER}/${IMAGE_NAME} \
                    --set image.tag=${IMAGE_TAG}
                """
            }
        }
    }
}
