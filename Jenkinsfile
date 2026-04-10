pipeline {
    agent any

    stages {
        stage('1. Code Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Container Image') {
            steps {
                sh 'docker build -t devsecops-app .'
            }
        }

        stage('3. DevSecOps Vulnerability Scan') {
            steps {
                // This scans the image and fails the pipeline if Severe vulnerabilities are found!
                sh 'trivy image --severity HIGH,CRITICAL devsecops-app'
            }
        }

        stage('4. Deploy to Servers') {
            steps {
                // Using scp and ssh to build and run on target servers seamlessly
                sh '''
                for HOST in "13.126.145.72" "43.205.254.82"; do
                    echo "Deploying to $HOST..."
                    scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null Dockerfile index.html ec2-user@$HOST:~
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ec2-user@$HOST "
                        docker stop my-app || true
                        docker rm my-app || true
                        docker build -t devsecops-app .
                        docker run -d -p 80:80 --name my-app devsecops-app
                        docker system prune -af || true
                    "
                done
                '''
            }
        }
    }
    
    post {
        always {
            // Aggressive cleanup to prevent your t3.micro disk full error from returning!
            sh 'docker system prune -af || true'
            cleanWs()
        }
    }
}
