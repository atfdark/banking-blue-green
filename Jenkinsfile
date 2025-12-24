pipeline {
    agent any

    environment {
        GREEN_HOST = "3.236.219.242"
        SSH_USER = "ec2-user"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Code checked out from GitHub'
            }
        }

        stage('Deploy to Green') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    scp -o StrictHostKeyChecking=no index.html \
                        ${SSH_USER}@${GREEN_HOST}:/var/www/html/index.html

                    ssh -o StrictHostKeyChecking=no \
                        ${SSH_USER}@${GREEN_HOST} 'sudo systemctl restart httpd'
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                sh "curl -f http://${GREEN_HOST}"
            }
        }
    }
}
