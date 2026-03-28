pipeline {
    agent any

    environment {
        // Domains from your assessment Task 2
        DEV_DOMAIN  = 'dev.xstrackers.top'
        PROD_DOMAIN = 'prod.xstrackers.top'
        
        DOCKER_USER = 'rahul699'
        IMAGE_NAME  = 'devops-assignment'
        IMAGE_TAG   = "${env.BUILD_ID}"
        FULL_IMAGE  = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout Code') {
            steps { checkout scm }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${FULL_IMAGE} ."
                sh "docker tag ${FULL_IMAGE} ${DOCKER_USER}/${IMAGE_NAME}:latest"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_VAR')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER_VAR --password-stdin"
                        sh "docker push ${FULL_IMAGE}"
                        sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { expression { return env.GIT_BRANCH == 'origin/dev' || env.GIT_BRANCH == 'dev' } }
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${DEV_DOMAIN} '
                        docker pull ${FULL_IMAGE}
                        docker stop app-container || true
                        docker rm app-container || true
                        docker run -d --name app-container -p 8081:8080 ${FULL_IMAGE}
                    '
                    """
                }
            }
        }

        stage('Wait for Approval') {
            when { expression { return env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main' } }
            steps {
                input message: "Deploy to Production Server?", ok: "Approve Deployment"
            }
        }

        stage('Deploy to Production') {
            when { expression { return env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main' } }
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${PROD_DOMAIN} '
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
        always { sh "docker logout || true" }
        success { echo "Deployment to ${env.GIT_BRANCH} successful!" }
    }
}
