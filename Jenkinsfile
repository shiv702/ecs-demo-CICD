pipeline {
    agent any

    environment {
        // === Adjust these for your account/region/service ===
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '970547338216'
        ECR_REPO_NAME  = 'ecs-cicd-demo'
        ECS_CLUSTER    = 'ecs-cicd-demo-cluster'
        ECS_SERVICE    = 'ecs-cicd-demo-service'

        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO_URI   = "${ECR_REGISTRY}/${ECR_REPO_NAME}"
    }

    options {
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build App (optional)') {
            steps {
                sh '''
                  echo "Installing dependencies..."
                  npm install
                  # npm test || echo "No tests configured"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def commitId = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = commitId
                }

                sh '''
                  echo "Building Docker image ${ECR_REPO_URI}:${IMAGE_TAG}"
                  docker build -t ${ECR_REPO_URI}:${IMAGE_TAG} .
                  docker tag ${ECR_REPO_URI}:${IMAGE_TAG} ${ECR_REPO_URI}:latest
                '''
            }
        }

        stage('Login to ECR & Push Image') {
            steps {
                // If you use IAM role on EC2, you can REMOVE this withCredentials block
                withCredentials([usernamePassword(
                    credentialsId: 'aws-jenkins-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {

                    sh '''
                      echo "Logging in to ECR..."
                      aws --version
                      aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                      echo "Pushing image tags ${IMAGE_TAG} and latest"
                      docker push ${ECR_REPO_URI}:${IMAGE_TAG}
                      docker push ${ECR_REPO_URI}:latest
                    '''
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                  echo "Triggering new ECS deployment..."
                  aws ecs update-service \
                    --cluster ${ECS_CLUSTER} \
                    --service ${ECS_SERVICE} \
                    --force-new-deployment \
                    --region ${AWS_REGION}

                  echo "Deployment triggered. ECS will pull the new :latest image."
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}
