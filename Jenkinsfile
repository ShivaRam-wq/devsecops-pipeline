pipeline {
    agent any
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        SLACK_WEBHOOK_URL = credentials('slack-webhook-url')
        SNS_TOPIC_ARN = 'arn:aws:sns:ap-south-1:522632170020:Pipeline-Alerts'
    }
    stages {
        stage('1. Code Checkout') { steps { checkout scm } }
        stage('2. SAST (SonarQube)') {
            steps {
                tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-admin-token', variable: 'SONAR_TOKEN_SECRET')]) {
                        sh "/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=devsecops-app -Dsonar.sources=. -Dsonar.token=$SONAR_TOKEN_SECRET -Dsonar.scanner.socketTimeout=600 || true"
                    }
                }
            }
        }
        stage('3. Build Image') { steps { sh 'docker build -t devsecops-app .' } }
        stage('4. Vulnerability Scan (Trivy)') { steps { sh 'trivy image --severity HIGH,CRITICAL devsecops-app' } }
        stage('5. Push to ECR') {
            steps {
                sh "aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 522632170020.dkr.ecr.ap-south-1.amazonaws.com"
                sh "docker tag devsecops-app:latest 522632170020.dkr.ecr.ap-south-1.amazonaws.com/devsecops-app:latest"
                sh "docker push 522632170020.dkr.ecr.ap-south-1.amazonaws.com/devsecops-app:latest"
            }
        }
        stage('6. Deploy (Ansible)') { steps { sh '/usr/local/bin/ansible-playbook -i inventory.ini deploy.yml' } }
        stage('7. DAST (OWASP ZAP)') {
            steps {
                sh "docker pull zaproxy/zap-stable"
                sh "docker run --rm -v \$(pwd):/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py -t http://13.127.141.111:30080 -r zap_report.html || true"
            }
        }
    }
    post {
        success {
            sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"✅ *Build Success:* #$BUILD_NUMBER Deployed to K8s\"}' \$SLACK_WEBHOOK_URL"
            sh "aws sns publish --topic-arn \$SNS_TOPIC_ARN --message 'Success: Build #$BUILD_NUMBER deployed!' --subject 'Pipeline Success' --region ap-south-1"
        }
        failure {
            sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"❌ *Build FAILED:* #$BUILD_NUMBER\"}' \$SLACK_WEBHOOK_URL"
            sh "aws sns publish --topic-arn \$SNS_TOPIC_ARN --message 'Failure: Build #$BUILD_NUMBER crashed!' --subject 'Pipeline Failure' --region ap-south-1"
        }
        always {
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            sh 'docker system prune -af || true'
            cleanWs()
        }
    }
}
