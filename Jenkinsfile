pipeline {
    agent any
    
    // Global dynamic variables for your AWS Vault!
    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = '522632170020.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPO = 'devsecops-app'
        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO}:latest"
    }

    stages {
        stage('1. Code Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Static Application Security Testing (SAST)') {
            environment {
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN_SECRET')]) {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=devsecops-app \
                            -Dsonar.sources=. \
                            -Dsonar.token=${SONAR_TOKEN_SECRET}"
                    }
                }
            }
        }

        stage('3. Build Container Image') {
            steps {
                sh 'docker build -t devsecops-app .'
            }
        }

        stage('4. Container Vulnerability Scan (Trivy)') {
            steps {
                sh 'trivy image --severity HIGH,CRITICAL devsecops-app'
            }
        }

        stage('5. Push to AWS ECR Private Vault') {
            steps {
                sh '''
                # 1. Login to AWS ECR securely (Jenkins IAM Role automates this!)
                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                
                # 2. Tag our scanned image with your personal ECR URI
                docker tag devsecops-app:latest ${IMAGE_URI}
                
                # 3. Push the artifact to vaulted cloud storage
                docker push ${IMAGE_URI}
                '''
            }
        }

        stage('6. Deploy to Production Servers') {
            steps {
                sh '''
                for HOST in "13.126.145.72" "43.205.254.82"; do
                    echo "Deploying to $HOST..."
                    
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ec2-user@$HOST "
                        # Servers authenticate to ECR using their IAM roles
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        
                        sudo systemctl stop nginx || true
                        docker stop my-app || true
                        docker rm my-app || true
                        
                        # Heavily optimized deployment: Just pull and launch the image from ECR!
                        docker pull ${IMAGE_URI}
                        docker run -d -p 80:80 --name my-app ${IMAGE_URI}
                        docker system prune -af || true
                    "
                done
                '''
            }
        }
    }
    
    post {
        always {
            sh 'docker system prune -af || true'
            cleanWs()
        }
    }
}
