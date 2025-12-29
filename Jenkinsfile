pipeline {
    agent any

    environment {
        // EC2 Hosts
        BLUE_HOST  = '35.175.103.106'       // BLUE EC2 Public IP
        GREEN_HOST = '3.237.27.218'        // GREEN EC2 Public IP

        SSH_USER = 'ec2-user'
        APP_DIR  = '/var/www/html'

        // S3
        S3_BUCKET = 'banking-bluegreen-artifacts'
        ARTIFACT  = "index-${BUILD_NUMBER}.html"

        // ALB & Target Groups
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

        stage('Detect Active Target Group') {
            steps {
                script {
                    def activeTG = sh(
                        script: """
                        aws elbv2 describe-listeners \
                          --listener-arn ${LISTENER_ARN} \
                          --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                          --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "🟢 Active Target Group: ${activeTG}"

                    if (activeTG == TG_BLUE) {
                        env.DEPLOY_TG = TG_GREEN
                        env.DEPLOY_HOST = GREEN_HOST
                        echo "🚀 Deploying to GREEN"
                    } else {
                        env.DEPLOY_TG = TG_BLUE
                        env.DEPLOY_HOST = BLUE_HOST
                        echo "🚀 Deploying to BLUE"
                    }
                }
            }
        }

        stage('Deploy to Inactive EC2 from S3') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh '''
                    chmod 600 $SSH_KEY
                    ssh -o StrictHostKeyChecking=no -i $SSH_KEY ${SSH_USER}@${DEPLOY_HOST} "
                        sudo mkdir -p /var/www/html &&
                        sudo chown -R ec2-user:ec2-user /var/www/html &&
                        aws s3 cp s3://${S3_BUCKET}/${ARTIFACT} /var/www/html/index.html
                    "
                    '''
                }
            }
        }

        stage('Switch Traffic to New Version') {
            steps {
                sh """
                aws elbv2 modify-listener \
                  --listener-arn ${LISTENER_ARN} \
                  --default-actions Type=forward,TargetGroupArn=${DEPLOY_TG}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Blue-Green Deployment Successful!"
        }

        failure {
            echo "❌ Deployment Failed. Rolling back traffic..."
            sh """
            aws elbv2 modify-listener \
              --listener-arn ${LISTENER_ARN} \
              --default-actions Type=forward,TargetGroupArn=${TG_BLUE}
            """
        }
    }
}
