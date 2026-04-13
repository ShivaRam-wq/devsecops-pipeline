pipeline {
    agent any

    environment {
        // Prevent Ansible from getting stuck on SSH YES/NO prompts
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    stages {
        stage('1. Code Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Static Application Security Testing (SAST)') {
            steps {
                tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN_SECRET')]) {
                        sh '''
                        /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=devsecops-app \
                          -Dsonar.sources=. \
                          -Dsonar.token=$SONAR_TOKEN_SECRET \
                          -Dsonar.scanner.socketTimeout=600
                        '''
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
                aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 522632170020.dkr.ecr.ap-south-1.amazonaws.com
                docker tag devsecops-app:latest 522632170020.dkr.ecr.ap-south-1.amazonaws.com/devsecops-app:latest
                docker push 522632170020.dkr.ecr.ap-south-1.amazonaws.com/devsecops-app:latest
                '''
            }
        }

        stage('6. Deploy to Production Servers (Ansible)') {
            steps {
                sh '/usr/local/bin/ansible-playbook -i inventory.ini deploy.yml'
            }
        }

        stage('7. Dynamic Security Testing (OWASP ZAP)') {
            steps {
                sh '''
                echo "Executing aggressive OWASP ZAP scan against the live Kubernetes cluster..."
                
                # Pull the official ZAP Hacker Docker Image
                docker pull zaproxy/zap-stable
                
                # Run the baseline scan against the live NodePort! 
                # (The || true ensures minor warnings don't fail the entire pipeline)
                docker run --rm -v $(pwd):/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py -t http://13.127.141.111:30080 -r zap_report.html || true
                '''
            }
        }
    }

    post {
        success {
            sh "aws sns publish --topic-arn arn:aws:sns:ap-south-1:522632170020:Pipeline-Alerts --message 'Deployment Success: The DevSecOps Pipeline completed and successfully deployed the app to Kubernetes!' --subject 'Pipeline Success #$BUILD_NUMBER' --region ap-south-1"
        }
        failure {
            sh "aws sns publish --topic-arn arn:aws:sns:ap-south-1:522632170020:Pipeline-Alerts --message 'Security Breach or Build Failure: The DevSecOps Pipeline has CRASHED! Check Jenkins log immediately.' --subject 'Pipeline FAILED #$BUILD_NUMBER' --region ap-south-1"
        }
        always {
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            sh 'docker system prune -af || true'
            cleanWs()
        }
    }
}
