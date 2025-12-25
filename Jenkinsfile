pipeline {
    agent any

    environment {
        SSH_USER         = 'ec2-user'
        GREEN_HOST       = '3.236.219.242'
        APP_DIR          = '/var/www/html'
        SSH_CRED_ID      = 'ec2-ssh-key'

        ALB_LISTENER_ARN = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:listener/app/banking-alb/XXXX/XXXX'
        TG_BLUE_ARN      = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-blue/XXXX'
        TG_GREEN_ARN     = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-green/XXXX'
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
                    sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY')
                ]) {
                    sh '''
                    chmod 600 $SSH_KEY
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
                  --listener-arn ${ALB_LISTENER_ARN} \
                  --default-actions Type=forward,ForwardConfig='{TargetGroups=[{TargetGroupArn=${TG_GREEN_ARN},Weight=100},{TargetGroupArn=${TG_BLUE_ARN},Weight=0}]}'
                """
            }
        }
    }

    post {
        failure {
            echo "Rolling back to BLUE..."
            sh """
            aws elbv2 modify-listener \
              --listener-arn ${ALB_LISTENER_ARN} \
              --default-actions Type=forward,ForwardConfig='{TargetGroups=[{TargetGroupArn=${TG_BLUE_ARN},Weight=100},{TargetGroupArn=${TG_GREEN_ARN},Weight=0}]}'
            """
        }
    }
}
