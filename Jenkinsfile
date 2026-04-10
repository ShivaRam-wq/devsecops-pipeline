pipeline {
    agent any
    stages {
        stage('Deploy to Server 2') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ec2-user@13.126.145.72 "echo '<h1><font color=red>Jenkins Deployment Server 2 ✅</font></h1>' | sudo tee /usr/share/nginx/html/index.html"
                '''
            }
        }
        
        stage('Deploy to Server 3') {
            steps {
                sh '''
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ec2-user@43.205.254.82 "echo '<h1><font color=blue>Jenkins Deployment Server 3 ✅</font></h1>' | sudo tee /usr/share/nginx/html/index.html"
                '''
            }
        }
    }
}
