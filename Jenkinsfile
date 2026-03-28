pipeline {
    agent any

    environment {
        // Domains as configured in your Route53/DNS
        DEV_HOST  = 'dev.xstrackers.top'
        PROD_HOST = 'prod.xstrackers.top'
        
        DOCKER_USER    = 'rahul699'
        IMAGE_NAME     = 'devops-assignment'
        IMAGE_TAG      = "${env.BUILD_ID}"
        FULL_IMAGE     = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE   = "${DOCKER_USER}/${IMAGE_NAME}:latest"
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
                    sh "docker tag ${FULL_IMAGE} ${LATEST_IMAGE}"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // Uses 'docker-hub-creds' created in Jenkins Credentials
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_VAR')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER_VAR --password-stdin"
                        sh "docker push ${FULL_IMAGE}"
                        sh "docker push ${LATEST_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            // Only runs when the branch is 'dev'
            when { expression { return env.GIT_BRANCH == 'origin/dev' || env.GIT_BRANCH == 'dev' } }
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${DEV_HOST} '
                        docker pull ${FULL_IMAGE}
                        docker stop app-container || true
                        docker rm app-container || true
                        # Port 8081 used here to avoid conflict with Jenkins on 8080
                        docker run -d --name app-container -p 8081:8080 ${FULL_IMAGE}
                    '
                    """
                }
            }
        }

        stage('Wait for Approval') {
            // Only runs when the branch is 'main'
            when { expression { return env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main' } }
            steps {
                input message: "Deploy Build #${IMAGE_TAG} to Production?", ok: "Approve Deployment"
            }
        }

        stage('Deploy to Production') {
            // Only runs when the branch is 'main'
            when { expression { return env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main' } }
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${PROD_HOST} '
                        docker pull ${FULL_IMAGE}
                        docker stop app-container || true
                        docker rm app-container || true
                        # Port 8080 is safe to use on the Production Server
                        docker run -d --name app-container -p 8080:8080 ${FULL_IMAGE}
                    '
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker logout || true"
            echo "Pipeline process for Build #${env.BUILD_ID} finished."
        }
        success {
            echo "SUCCESS: Deployment completed successfully."
        }
        failure {
            echo "FAILURE: Deployment failed. Please review Jenkins console output."
        }
    }
}
