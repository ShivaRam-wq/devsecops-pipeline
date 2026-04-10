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
                            -Dsonar.token=${SONAR_TOKEN_SECRET} \
                            -Dsonar.ws.timeout=300"
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
                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                docker tag devsecops-app:latest ${IMAGE_URI}
                docker push ${IMAGE_URI}
                '''
            }
        }

        stage('6. Deploy to Production Servers (Ansible)') {
            steps {
                // This one clean line completely coordinates the complex deployment across all cloud servers using your new deploy.yml playbook!
                sh '/usr/local/bin/ansible-playbook -i inventory.ini deploy.yml'
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
