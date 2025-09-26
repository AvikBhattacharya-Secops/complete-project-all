pipeline {
    agent any

    environment {
        // Jenkins credentials
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-creds')            // DockerHub
        AWS_CREDENTIALS = credentials('aws-credientials')                  // AWS credentials for ECR
        GIT_CREDENTIALS = credentials('GitAccess')                         // GitHub access
        ARGOCD_CREDENTIALS = credentials('argocd')                         // ArgoCD access

        // AWS and DockerHub details
        REGION = 'ap-south-1'
        IMAGE_TAG = "${BUILD_NUMBER}"
        AWS_ACCOUNT_ID = '439110395780'
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/my-repo"
        DOCKERHUB_REPO = 'avikbhattacharya056/my-calculator-image'
        ARGOCD_SERVER = '13.235.24.48:32506'
    }

    stages {

        stage('Checkout') {
            steps {
                git credentialsId: "${GIT_CREDENTIALS}", url: 'https://github.com/AvikBhattacharya-Secops/complete-project-all.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    echo "Logging in and pushing to DockerHub..."
                    sh """
                        echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    echo "Logging in and pushing to AWS ECR..."
                    withEnv(["AWS_ACCESS_KEY_ID=${AWS_CREDENTIALS_USR}", "AWS_SECRET_ACCESS_KEY=${AWS_CREDENTIALS_PSW}"]) {
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com
                            docker push ${ECR_REPO}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                script {
                    echo "Updating Helm values.yaml..."
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
                    echo "Committing and pushing Helm changes to GitHub..."
                    sh """
                        git config user.email 'ci@jenkins.com'
                        git config user.name 'Jenkins CI'
                        git add helm/values.yaml
                        git commit -m 'Update image tag to ${IMAGE_TAG}' || echo 'No changes to commit'
                        git push origin main
                    """
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    echo "Triggering ArgoCD sync..."
                    withCredentials([usernamePassword(credentialsId: 'argocd', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PASSWORD')]) {
                        sh """
                            argocd login ${ARGOCD_SERVER} --username $ARGOCD_USER --password $ARGOCD_PASSWORD --insecure || echo 'ArgoCD login failed or already logged in'
                            argocd app sync calculator-app || echo 'ArgoCD sync failed or app not found'
                        """
                    }
                }
            }
        }
    }
}
