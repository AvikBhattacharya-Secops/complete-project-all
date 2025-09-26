pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred-id')  // Jenkins credentials ID
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        REGION = 'us-east-1' // update your AWS region
        ECR_REPO = '123456789012.dkr.ecr.us-east-1.amazonaws.com/calculator' // your ECR URL
        DOCKERHUB_REPO = 'yourdockerhub/calculator'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-user/complete-project-all.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $DOCKERHUB_REPO:$IMAGE_TAG .'
                    sh 'docker tag $DOCKERHUB_REPO:$IMAGE_TAG $ECR_REPO:$IMAGE_TAG'
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"
                    sh "docker push $DOCKERHUB_REPO:$IMAGE_TAG"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REPO"
                    sh "docker push $ECR_REPO:$IMAGE_TAG"
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                script {
                    sh """
                        sed -i 's|repository:.*|repository: $DOCKERHUB_REPO|' helm/values.yaml
                        sed -i 's|tag:.*|tag: $IMAGE_TAG|' helm/values.yaml
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
                    sh "git commit -m 'Update image tag to $IMAGE_TAG'"
                    sh "git push origin main"
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    // Optional: You can setup ArgoCD webhook or use CLI to sync
                    // If ArgoCD CLI is installed and accessible
                    sh "argocd app sync calculator-app"
                }
            }
        }
    }
}
