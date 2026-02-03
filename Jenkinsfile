pipeline {
    agent any

    tools {
        maven 'maven9'
    }

    environment {
        AWS_REGION = 'eu-north-1'
        CLUSTER_NAME = 'my-eks-cluster'

        AWS_ACCOUNT_ID = '162343471712'
        ECR_REPO = 'my-app'

        IMAGE_TAG = "${BUILD_NUMBER}"
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {

        stage('0. Test AWS CLI') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    echo "Testing AWS CLI with credentials..."
                    aws --version
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('1. Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Maven Build') {
            steps {
                sh '''
                echo "Running Maven build..."
                mvn clean package -DskipTests
                '''
            }
        }

        stage('3. Build Docker Image') {
            steps {
                sh '''
                echo "Building Docker image..."
                docker build -t ${ECR_URI}:${IMAGE_TAG} .
                '''
            }
        }

        stage('4. Ensure ECR Repository') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    echo "Checking if ECR repository exists..."
                    if ! aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} > /dev/null 2>&1; then
                        echo "Repository does not exist. Creating..."
                        aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                    else
                        echo "Repository already exists."
                    fi
                    '''
                }
            }
        }

        stage('5. Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    echo "Logging in to ECR..."
                    aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_URI}
                    '''
                }
            }
        }

        stage('6. Push Docker Image to ECR') {
            steps {
                sh '''
                echo "Pushing Docker image to ECR..."
                docker push ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('7. Connect to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                    echo "Updating kubeconfig for EKS..."
                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${CLUSTER_NAME}
                    '''
                }
            }
        }

        stage('8. Deploy to EKS') {
            steps {
                sh '''
                echo "Deploying to EKS..."
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
