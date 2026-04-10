pipeline {
    agent any

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
                    sh "${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=devsecops-app \
                        -Dsonar.sources=. \
                        -Dsonar.token=${env.SONAR_TOKEN}" 
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

        stage('5. Deploy to Servers') {
            steps {
                sh '''
                for HOST in "13.126.145.72" "43.205.254.82"; do
                    echo "Deploying to $HOST..."
                    scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null Dockerfile index.html ec2-user@$HOST:~
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ec2-user@$HOST "
                        sudo systemctl stop nginx || true
                        sudo systemctl disable nginx || true
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
            sh 'docker system prune -af || true'
            cleanWs()
        }
    }
}
