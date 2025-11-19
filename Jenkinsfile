pipeline {
    agent any

    // 1) Parameter comes from job configuration (dev / qa / prod)
    parameters {
        choice(
            name: 'TARGET_ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Environment to deploy to'
        )
    }

    options {
        timestamps()
    }

    stages {

        stage('Init env') {
            steps {
                script {
                    // 2) Base values from Jenkins global env (or fallback)
                    env.AWS_REGION     = env.AWS_REGION ?: 'ap-south-1'
                    env.AWS_ACCOUNT_ID = env.AWS_ACCOUNT_ID ?: '970547338216'

                    // 3) Environment-specific naming
                    def envName = params.TARGET_ENV ?: 'dev'
                    env.DEPLOY_ENV   = envName

                    // Example naming convention; change if yours is different
                    env.ECR_REPO_NAME = "ecs-cicd-demo-${envName}"
                    env.ECS_CLUSTER   = "ecs-cicd-demo-cluster-${envName}"
                    env.ECS_SERVICE   = "ecs-cicd-demo-service-${envName}"

                    env.ECR_REGISTRY  = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                    env.ECR_REPO_URI  = "${env.ECR_REGISTRY}/${env.ECR_REPO_NAME}"

                    echo """
                    ===== Deployment configuration =====
                    TARGET_ENV   : ${env.DEPLOY_ENV}
                    AWS_ACCOUNT  : ${env.AWS_ACCOUNT_ID}
                    AWS_REGION   : ${env.AWS_REGION}
                    ECR_REPO_URI : ${env.ECR_REPO_URI}
                    ECS_CLUSTER  : ${env.ECS_CLUSTER}
                    ECS_SERVICE  : ${env.ECS_SERVICE}
                    ====================================
                    """
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // stage('Build App (optional)') {
        //     steps {
        //         sh '''
        //           echo "Installing dependencies..."
        //           npm install
        //           # npm test || echo "No tests configured"
        //         '''
        //     }
        // }

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
                      echo "Configuring AWS CLI from Jenkins credentials..."
                      aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                      aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                      aws configure set default.region ${AWS_REGION}

                      echo "Logging in to ECR..."
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
                  echo "Triggering new ECS deployment to ${DEPLOY_ENV}..."
                  aws ecs update-service \
                    --cluster ${ECS_CLUSTER} \
                    --service ${ECS_SERVICE} \
                    --force-new-deployment \
                    --region ${AWS_REGION}

                  echo "Deployment triggered. ECS in ${DEPLOY_ENV} will pull the new :latest image."
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
