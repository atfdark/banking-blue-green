pipeline {
    agent any

    environment {
        SSH_USER = 'ec2-user'
        GREEN_HOST = '3.236.219.242'
        APP_DIR = '/var/www/html'
        SSH_CRED_ID = 'ec2-ssh-key'

        LISTENER_ARN = 'arn:aws:elasticloadbalancing:us-east-1:647258324843:listener/app/banking-alb/67ba21a6e538e97a/24b61834ce28b868'
        TG_BLUE = 'arn:aws:elasticloadbalancing:us-east-1:647258324843:targetgroup/tg-blue/28069bd78d9fb8c7'
        TG_GREEN = 'arn:aws:elasticloadbalancing:us-east-1:647258324843:targetgroup/tg-green/6d97b09d5a26c8ab'
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

                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ec2-user@${GREEN_HOST} \
                    "sudo mkdir -p ${APP_DIR} && sudo chown -R ec2-user:ec2-user ${APP_DIR}"

                    scp -i $SSH_KEY -o StrictHostKeyChecking=no index.html \
                    ec2-user@${GREEN_HOST}:${APP_DIR}/index.html
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
