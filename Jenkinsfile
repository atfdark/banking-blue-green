pipeline {
    agent any

    environment {
        GREEN_HOST = "13.220.140.10"
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
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ec2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                    chmod 600 $SSH_KEY

                    scp -i $SSH_KEY -o StrictHostKeyChecking=no index.html \
                        ${SSH_USER}@${GREEN_HOST}:/var/www/html/index.html

                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no \
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
