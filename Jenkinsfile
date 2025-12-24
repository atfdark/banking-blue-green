pipeline {
    agent any

    environment {
        SSH_USER          = 'ec2-user'
        GREEN_HOST        = '3.236.219.242'   // Green EC2 public IP
        APP_DIR           = '/var/www/html'
        SSH_CRED_ID       = 'ec2-ssh-key'        // Jenkins SSH credential ID

        ALB_LISTENER_ARN  = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:listener/app/banking-alb/XXXX/XXXX'
        TG_BLUE_ARN       = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-blue/XXXX'
        TG_GREEN_ARN      = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-green/XXXX'
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
                    scp -i $SSH_KEY -o StrictHostKeyChecking=no index.html \
                        ec2-user@${GREEN_HOST}:/var/www/html/index.html
                    '''
                }
            }
        }

        stage('Health Check GREEN') {
            steps {
                script {
                    echo "Waiting for GREEN to stabilize..."
                    sleep 15

                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://${ALB_DNS}",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {
                        error "GREEN health check FAILED"
                    }

                    echo "GREEN health check PASSED"
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
        failure {
            echo "❌ Deployment failed. Rolling back to BLUE..."

            sh """
            aws elbv2 modify-listener \
              --listener-arn ${LISTENER_ARN} \
              --default-actions Type=forward,TargetGroupArn=${TG_BLUE}
            """
        }

        success {
            echo "✅ Deployment successful. GREEN is live!"
        }
    }
}
