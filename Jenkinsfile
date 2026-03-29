pipeline {
    agent any

    environment {
        DEV_HOST  = 'dev.xstrackers.top'
        PROD_HOST = 'prod.xstrackers.top'

        DOCKER_USER  = 'rahul699'
        IMAGE_NAME   = 'devops-assignment'
        IMAGE_TAG    = "${env.BUILD_ID}"
        FULL_IMAGE   = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE = "${DOCKER_USER}/${IMAGE_NAME}:latest"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Image: ${FULL_IMAGE}"
                    sh "docker build -t ${FULL_IMAGE} ."
                    sh "docker tag ${FULL_IMAGE} ${LATEST_IMAGE}"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        passwordVariable: 'DOCKER_PASS',
                        usernameVariable: 'DOCKER_USER_VAR'
                    )]) {
                        sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER_VAR --password-stdin
                        docker push ${FULL_IMAGE}
                        docker push ${LATEST_IMAGE}
                        docker logout
                        """
                    }
                }
            }
        }

        // ✅ DEV Deployment
        stage('Deploy to Dev') {
            when { branch 'dev' }
            steps {
                echo "Deploying to DEV..."
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${DEV_HOST} '
                        docker pull ${LATEST_IMAGE}
                        docker stop app-container || true
                        docker rm app-container || true
                        docker run -d --restart unless-stopped --name app-container -p 8081:8080 ${LATEST_IMAGE}
                    '
                    """
                }
            }
        }

        // ✅ Approval before PROD
        stage('Approval for Production') {
            when { branch 'main' }
            steps {
                input message: "Deploy Build #${IMAGE_TAG} to Production?", ok: "Deploy"
            }
        }

        // ✅ PROD Deployment
        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                echo "Deploying to PROD..."
                sshagent(['ec2-ssh-key']) {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        passwordVariable: 'DOCKER_PASS',
                        usernameVariable: 'DOCKER_USER_VAR'
                    )]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_HOST} '
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER_VAR" --password-stdin
                            docker pull ${LATEST_IMAGE}
                            docker stop app-container || true
                            docker rm app-container || true
                            docker run -d --restart unless-stopped --name app-container -p 8080:8080 ${LATEST_IMAGE}
                        '
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Build #${env.BUILD_ID} completed."
        }
        success {
            echo "SUCCESS: Deployment completed."
        }
        failure {
            echo "FAILURE: Check logs in Jenkins."
        }
    }
}
