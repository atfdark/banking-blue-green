pipeline {
    agent any

    environment {
        SSH_USER = 'ec2-user'
        GREEN_HOST = '3.236.219.242'     // GREEN EC2 PUBLIC IP
        APP_DIR = '/var/www/html'
        SSH_CRED_ID = 'ec2-ssh-key'

        ALB_LISTENER_ARN = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:listener/app/banking-alb/XXXX/XXXX'
        TG_BLUE_ARN  = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-blue/XXXX'
        TG_GREEN_ARN = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-green/XXXX'
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
                        "sudo rm -rf /var/www/html/*"

                    scp -i $SSH_KEY -o StrictHostKeyChecking=no index.html \
                        ec2-user@${GREEN_HOST}:/var/www/html/index.html

                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ec2-user@${GREEN_HOST} \
                        "sudo systemctl restart httpd"
                    '''
                }
            }
        }

        stage('Health Check GREEN') {
            steps {
                script {
                    echo "Waiting for GREEN to stabilize..."
                    sleep 10

                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://${GREEN_HOST}",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {
                        error "❌ GREEN health check FAILED"
                    }

                    echo "✅ GREEN health check PASSED"
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
            echo "❌ Deployment failed. Rolling back to BLUE..."

            sh """
            aws elbv2 modify-listener \
              --listener-arn ${ALB_LISTENER_ARN} \
              --default-actions Type=forward,ForwardConfig='{TargetGroups=[{TargetGroupArn=${TG_BLUE_ARN},Weight=100},{TargetGroupArn=${TG_GREEN_ARN},Weight=0}]}'
            """
        }

        success {
            echo "✅ Deployment successful. GREEN is LIVE!"
        }
    }
}
