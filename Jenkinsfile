pipeline {
    agent any

    environment {
        SSH_USER = 'ec2-user'
        GREEN_HOST = '3.236.219.242'
        APP_DIR = '/var/www/html'
        SSH_CRED_ID = 'ec2-ssh-key'

        LISTENER_ARN = 'PASTE_LISTENER_ARN_HERE'
        TG_BLUE = 'PASTE_TG_BLUE_ARN'
        TG_GREEN = 'PASTE_TG_GREEN_ARN'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git credentialsId: 'github-creds',
                    url: 'https://github.com/atfdark/banking-blue-green.git',
                    branch: 'main'
            }
        }

        stage('Deploy to GREEN') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh '''
                    chmod 600 $SSH_KEY
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ec2-user@${GREEN_HOST} "mkdir -p /var/www/html"
                    scp -i $SSH_KEY -o StrictHostKeyChecking=no index.html ec2-user@${GREEN_HOST}:/var/www/html/index.html
                    '''
                }
            }
        }

        stage('Switch Traffic to GREEN') {
            steps {
                sh """
                aws elbv2 modify-listener \
                  --listener-arn ${LISTENER_ARN} \
                  --default-actions Type=forward,TargetGroupArn=${TG_GREEN}
                """
            }
        }
    }

    post {
        success {
            echo "✅ GREEN deployment successful!"
        }

        failure {
            echo "❌ Deployment failed. Rolling back to BLUE..."
            sh """
            aws elbv2 modify-listener \
              --listener-arn ${LISTENER_ARN} \
              --default-actions Type=forward,TargetGroupArn=${TG_BLUE}
            """
        }
    }
}
