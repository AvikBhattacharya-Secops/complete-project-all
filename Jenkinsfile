pipeline {
    agent any

    environment {
        // Jenkins credentials
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-creds')            // DockerHub
        AWS_CREDENTIALS = credentials('aws-credientials')                  // AWS credentials for ECR
        GIT_CREDENTIALS = credentials('GitAccess')                         // GitHub

        // AWS and DockerHub details
        REGION = 'ap-south-1'
        IMAGE_TAG = "${BUILD_NUMBER}"
        AWS_ACCOUNT_ID = '439110395780'
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/my-repo"
        DOCKERHUB_REPO = 'avikbhattacharya056/my-calculator-image'
    }

    stages {

        stage('Checkout') {
            steps {
                git credentialsId: 'GitAccess', url: 'https://github.com/AvikBhattacharya-Secops/complete-project-all.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
                    sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                script {
                    sh """
                        sed -i 's|repository:.*|repository: ${DOCKERHUB_REPO}|' helm/values.yaml
                        sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' helm/values.yaml
                    """
                }
            }
        }

        stage('Push Helm Changes') {
            steps {
                script {
                    sh "git config user.email 'ci@jenkins.com'"
                    sh "git config user.name 'Jenkins CI'"
                    sh "git add helm/values.yaml"
                    sh "git commit -m 'Update image tag to ${IMAGE_TAG}' || echo 'No changes to commit'"
                    sh "git push origin main"
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    // Optional: ArgoCD CLI must be configured in Jenkins agent
                    sh "argocd app sync calculator-app || echo 'ArgoCD sync failed or not configured'"
                }
            }
        }
    }
}
