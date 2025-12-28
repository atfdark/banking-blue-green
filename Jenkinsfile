pipeline {
    agent any

    environment {
        GREEN_HOST = '3.237.27.218'
        SSH_USER   = 'ec2-user'
        APP_DIR   = '/var/www/html'

        S3_BUCKET = 'banking-bluegreen-artifacts'
        ARTIFACT  = "index-${BUILD_NUMBER}.html"

        LISTENER_ARN = 'arn:aws:elasticloadbalancing:us-east-1:647258324843:listener/app/banking-alb/67ba21a6e538e97a/24b61834ce28b868'
        TG_BLUE  = 'arn:aws:elasticloadbalancing:us-east-1:647258324843:targetgroup/tg-blue/28069bd78d9fb8c7'
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

        stage('Prepare Artifact') {
            steps {
                sh '''
                  cp index.html ${ARTIFACT}
                '''
            }
        }

        stage('Upload Artifact to S3') {
            steps {
                sh '''
                  aws s3 cp ${ARTIFACT} s3://${S3_BUCKET}/${ARTIFACT}
                '''
            }
        }

        stage('Deploy to GREEN from S3') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh '''
                    chmod 600 $SSH_KEY
                    ssh -i $SSH_KEY ec2-user@${GREEN_HOST} "
                      sudo mkdir -p /var/www/html &&
                      sudo chown -R ec2-user:ec2-user /var/www/html &&
                      aws s3 cp s3://${S3_BUCKET}/${ARTIFACT} /var/www/html/index.html
                    "
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
            echo "✅ Deployment successful via S3 artifact!"
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
