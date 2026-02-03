pipeline {
    agent any

    environment {
        AWS_REGION = 'eu-north-1'
        AWS_ACCOUNT_ID = '162343471712'
        ECR_REPO = 'my-app'
        IMAGE_TAG = 'latest'
        K8S_DEPLOYMENT = 'my-app'
        EKS_CLUSTER_NAME = 'my-eks-cluster'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/7sanjay/eks.git', credentialsId: 'nexus-creds'
            }
        }

        stage('Maven Build') {
            steps {
                sh '''
                    echo "Running Maven build..."
                    mvn clean package -DskipTests
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Ensure ECR Repository') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        echo "Checking if ECR repository exists..."
                        aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} || \
                        aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        echo "Logging in to ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh '''
                    echo "Pushing Docker image to ECR..."
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Update kubeconfig') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        echo "Updating kubeconfig for EKS..."
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    echo "Deploying to EKS..."
                    kubectl set image deployment/${K8S_DEPLOYMENT} ${K8S_DEPLOYMENT}=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG} --record
                    kubectl rollout status deployment/${K8S_DEPLOYMENT}
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment completed successfully!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
