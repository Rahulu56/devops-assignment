pipeline {
    agent any

    environment {
        // Replace these with your actual EC2 Public IPs
        DEV_SERVER_IP  = 'YOUR_DEV_EC2_PUBLIC_IP'
        PROD_SERVER_IP = 'YOUR_PROD_EC2_PUBLIC_IP'
        
        DOCKER_USER    = 'rahul699'
        IMAGE_NAME     = 'devops-assignment'
        IMAGE_TAG      = "${env.BUILD_ID}"
        FULL_IMAGE     = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Jenkins automatically pulls the code from the linked Git repo
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Image: ${FULL_IMAGE}"
                    sh "docker build -t ${FULL_IMAGE} ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // This uses the 'docker-hub-creds' ID created in Jenkins UI
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_VAR')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER_VAR --password-stdin"
                        sh "docker push ${FULL_IMAGE}"
                        // Also push a 'latest' tag for convenience
                        sh "docker tag ${FULL_IMAGE} ${DOCKER_USER}/${IMAGE_NAME}:latest"
                        sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { branch 'dev' } // Only runs when code is pushed to 'dev' branch
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${DEV_SERVER_IP} '
                        docker pull ${FULL_IMAGE}
                        docker stop app-container || true
                        docker rm app-container || true
                        docker run -d --name app-container -p 8080:8080 ${FULL_IMAGE}
                    '
                    """
                }
            }
        }

        stage('Wait for Approval') {
            when { branch 'main' } // Only runs for the 'main' branch
            steps {
                input message: "Deploy to Production Server?", ok: "Approve Deployment"
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' } // Only runs for the 'main' branch
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${PROD_SERVER_IP} '
                        docker pull ${FULL_IMAGE}
                        docker stop app-container || true
                        docker rm app-container || true
                        docker run -d --name app-container -p 8080:8080 ${FULL_IMAGE}
                    '
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker logout"
            echo "Pipeline finished."
        }
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed. Check logs."
        }
    }
}
