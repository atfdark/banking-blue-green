pipeline {
    agent any

    environment {
        SSH_USER          = 'ec2-user'
        GREEN_HOST        = '3.236.219.242'   // Green EC2 public IP
        APP_DIR           = '/var/www/html'
        SSH_CRED_ID       = 'bank-key'        // Jenkins SSH credential ID

        ALB_LISTENER_ARN  = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:listener/app/banking-alb/XXXX/XXXX'
        TG_BLUE_ARN       = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-blue/XXXX'
        TG_GREEN_ARN      = 'arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/tg-green/XXXX'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                echo "Code pulled from GitHub"
            }
        }

        stage('Deploy to GREEN') {
            steps {
                sshagent(credentials: [SSH_CRED_ID]) {
                    sh '''
                        echo "Deploying to GREEN server"
                        ssh -o StrictHostKeyChecking=no $SSH_USER@$GREEN_HOST "sudo mkdir -p ${APP_DIR} && sudo chown -R ec2-user:ec2-user ${APP_DIR}"
                        scp -o StrictHostKeyChecking=no index.html $SSH_USER@$GREEN_HOST:${APP_DIR}/index.html
                        ssh -o StrictHostKeyChecking=no $SSH_USER@$GREEN_HOST "sudo systemctl restart httpd"
                    '''
                }
            }
        }

        stage('Health Check GREEN') {
            steps {
                script {
                    echo "Waiting for application startup..."
                    sleep 15

                    def status = sh(
                        script: "curl -o /dev/null -s -w '%{http_code}' http://${GREEN_HOST}",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {
                        error "Health check failed on GREEN (HTTP ${status})"
                    }

                    echo "GREEN environment is healthy"
                }
            }
        }

        stage('Switch Traffic to GREEN') {
            steps {
                echo "Switching ALB traffic to GREEN"

                sh """
                aws elbv2 modify-listener \
                  --listener-arn ${ALB_LISTENER_ARN} \
                  --default-actions Type=forward,ForwardConfig='{
                    "TargetGroups":[
                      {"TargetGroupArn":"${TG_GREEN_ARN}","Weight":100},
                      {"TargetGroupArn":"${TG_BLUE_ARN}","Weight":0}
                    ]
                  }'
                """
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed. BLUE environment remains live."
        }

        success {
            echo "✅ Deployment successful. Traffic switched to GREEN."
        }
    }
}
