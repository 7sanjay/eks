pipeline {
    agent any

    environment {
        AWS_REGION = "eu-north-1"
        ECR_REPO = "my-app"
        IMAGE_TAG = "latest"
        CLUSTER_NAME = "my-eks-cluster"
        KUBECTL = "/usr/local/bin/kubectl" // make sure kubectl is installed
    }

    stages {
        stage('Test AWS CLI') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        echo "Testing AWS CLI..."
                        aws --version
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Ensure ECR Repository') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        if ! aws ecr describe-repositories --repository-names $ECR_REPO --region $AWS_REGION; then
                            echo "Creating ECR repository..."
                            aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION
                        else
                            echo "ECR repository exists."
                        fi
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
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO
                    '''
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh """
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                    docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Create or Update EKS Cluster') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        # Check if cluster exists
                        CLUSTER_EXISTS=$(aws eks list-clusters --region $AWS_REGION | grep $CLUSTER_NAME || true)
                        if [ -z "$CLUSTER_EXISTS" ]; then
                            echo "Creating EKS cluster..."
                            aws eks create-cluster \
                                --name $CLUSTER_NAME \
                                --region $AWS_REGION \
                                --kubernetes-version 1.30 \
                                --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/EKS-Root-Role \
                                --resources-vpc-config subnetIds=<SUBNET_IDS>,securityGroupIds=<SG_IDS>
                            echo "Waiting for cluster to be ACTIVE..."
                            aws eks wait cluster-active --name $CLUSTER_NAME --region $AWS_REGION
                        else
                            echo "EKS cluster already exists."
                        fi
                    '''
                }
            }
        }

        stage('Update kubeconfig') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    ${KUBECTL} set image deployment/my-app my-app=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG} || \
                    ${KUBECTL} apply -f k8s/deployment.yaml
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment succeeded!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
