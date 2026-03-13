pipeline {
    agent any
    environment {
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:us-east-1:774222655403:listener/app/my-Blue-Green-ALB/952df24cae747f6c/7ce76dd9e8b3d252'  // ALB console madhun copy
        BLUE_TG_ARN  = 'arn:aws:elasticloadbalancing:us-east-1:774222655403:targetgroup/Tg-Blue/13370a766c96fd69'
        GREEN_TG_ARN = 'arn:aws:elasticloadbalancing:us-east-1:774222655403:targetgroup/Tg-Green/04bf97860d34905b'
        BLUE_IP      = '172.31.27.61'
        GREEN_IP     = '172.31.28.223'
    }
    stages {
        stage('1. Pull code from GitHub') {
            steps {
                git branch: 'main', url: 'https://github.com/Abhinandan-58/Production-Ready-Blue-Green-Deployment-project.git'
            }
        }
        stage('2. Find Inactive Environment') {
            steps {
                script {
                    def currentTG = sh(script: "aws elbv2 describe-listeners --listener-arn ${LISTENER_ARN} --query 'Listeners[0].DefaultActions[0].ForwardConfig.TargetGroups[0].TargetGroupArn' --output text", returnStdout: true).trim()
                    if (currentTG == BLUE_TG_ARN) {
                        env.INACTIVE_TG = GREEN_TG_ARN
                        env.INACTIVE_IP = GREEN_IP
                        env.ACTIVE_TG = BLUE_TG_ARN
                    } else {
                        env.INACTIVE_TG = BLUE_TG_ARN
                        env.INACTIVE_IP = BLUE_IP
                        env.ACTIVE_TG = GREEN_TG_ARN
                    }
                }
            }
        }
        stage('3. Deploy to Inactive Environment') {
            steps {
                sh """
                    scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/key.pem index.html ec2-user@${INACTIVE_IP}:/var/www/html/
                    ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/key.pem ec2-user@${INACTIVE_IP} 'sudo systemctl restart httpd'
                """
            }
        }
        stage('4. Health Check') {
            steps {
                script {
                    def health = sh(script: "aws elbv2 describe-target-health --target-group-arn ${INACTIVE_TG} --query 'TargetHealthDescriptions[0].TargetHealth.State' --output text", returnStdout: true).trim()
                    if (health != "healthy") {
                        error("Health check failed!")
                    }
                }
            }
        }
        stage('5. Switch ALB Traffic') {
            steps {
                sh """
                    aws elbv2 modify-listener --listener-arn ${LISTENER_ARN} \
                    --default-actions '[{"Type":"forward","ForwardConfig":{"TargetGroups":[{"TargetGroupArn":"${INACTIVE_TG}","Weight":1}]}}]'
                """
            }
        }
    }
    post {
        failure {
            echo "Rollback triggered!"
            sh """
                aws elbv2 modify-listener --listener-arn ${LISTENER_ARN} \
                --default-actions '[{"Type":"forward","ForwardConfig":{"TargetGroups":[{"TargetGroupArn":"${ACTIVE_TG}","Weight":1}]}}]'
            """
        }
        success {
            echo "Deployment successful! Traffic switched."
        }
    }
}