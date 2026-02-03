pipeline {
    agent any

    tools {
        maven 'maven9'
    }

    environment {
        AWS_REGION = 'eu-north-1'
        CLUSTER_NAME = 'my-eks-cluster'

        AWS_ACCOUNT_ID = '123456789012'
        ECR_REPO = 'my-app'

        IMAGE_TAG = "${BUILD_NUMBER}"
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {

        stage('1. Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('3. Build Docker Image') {
            steps {
                sh 'docker build -t my-app:${IMAGE_TAG} .'
            }
        }

        stage('4. Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                      aws ecr get-login-password --region eu-north-1 \
                      | docker login --username AWS --password-stdin ${ECR_URI}
                    '''
                }
            }
        }

        stage('5. Push Image to ECR') {
            steps {
                sh '''
                  docker tag my-app:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
                  docker push ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('6. Connect to EKS') {
            steps {
                sh '''
                  aws eks update-kubeconfig \
                    --region eu-north-1 \
                    --name my-eks-cluster
                '''
            }
        }

        stage('7. Deploy to EKS') {
            steps {
                sh '''
                  kubectl set image deployment/my-app \
                    my-app=${ECR_URI}:${IMAGE_TAG}

                  kubectl rollout status deployment/my-app
                '''
            }
        }
    }

    post {
        success {
            echo '🎉 Deployment Successful!'
        }
        failure {
            echo '❌ Deployment Failed'
        }
    }
}
