pipeline {
    agent any

    tools {
        maven 'maven9'  // Ensure Maven tool is configured in Jenkins
    }

    environment {
        AWS_REGION     = 'eu-north-1'
        CLUSTER_NAME   = 'my-eks-cluster'
        AWS_ACCOUNT_ID = '162343471712'
        ECR_REPO       = 'my-app'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        ECR_URI        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        PATH           = "/usr/local/bin:${env.PATH}" // Ensures Jenkins finds aws CLI
    }

    stages {

        stage('0. Test AWS CLI') {
            steps {
                sh '''
                    echo "Testing AWS CLI..."
                    aws --version
                    aws sts get-caller-identity
                '''
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

        stage('4. Login to ECR') {
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

        stage('5. Push Docker Image to ECR') {
            steps {
                sh '''
                    echo "Pushing Docker image to ECR..."
                    docker push ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('6. Connect to EKS') {
            steps {
                sh '''
                    echo "Updating kubeconfig for EKS..."
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                '''
            }
        }

        stage('7. Deploy to EKS') {
            steps {
                sh '''
                    echo "Deploying to EKS..."
                    # Update deployment image if it exists, else apply YAML
                    if kubectl get deployment my-app; then
                        kubectl set image deployment/my-app my-app=${ECR_URI}:${IMAGE_TAG} --record
                    else
                        kubectl apply -f k8s/deployment.yaml
                    fi

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
