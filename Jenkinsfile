pipeline {
    agent any
    environment {
        ALB_NAME = 'My-Blue-Green-ALB'
        BLUE_TG  = 'TG-Blue'
        GREEN_TG = 'TG-Green'
    }
    stages {
        stage('Pull Code from GitHub') {
            steps {
                git branch: 'main', url: 'https://github.com/Abhinandan-58/Production-Ready-Blue-Green-Deployment-project.git'
            }
        }
        stage('Determine Inactive Environment') {
            steps {
                script {
                    def listenerArn = sh(script: """
                        aws elbv2 describe-listeners \
                        --load-balancer-arn \$(aws elbv2 describe-load-balancers --names ${ALB_NAME} --query 'LoadBalancers[0].LoadBalancerArn' --output text) \
                        --query 'Listeners[0].ListenerArn' --output text
                    """, returnStdout: true).trim()

                    def currentTG = sh(script: """
                        aws elbv2 describe-listeners --listener-arn ${listenerArn} \
                        --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text
                    """, returnStdout: true).trim()

                    if (currentTG.contains('blue')) {
                        env.INACTIVE_ENV = 'Green'
                        env.ACTIVE_TG    = 'blue-tg'
                        env.INACTIVE_TG  = 'green-tg'
                    } else {
                        env.INACTIVE_ENV = 'Blue'
                        env.ACTIVE_TG    = 'green-tg'
                        env.INACTIVE_TG  = 'blue-tg'
                    }
                    echo "Inactive environment: ${env.INACTIVE_ENV}"
                }
            }
        }
        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    def inactiveInstanceName = env.INACTIVE_ENV.toLowerCase()
                    def publicIp = sh(script: """
                        aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=${inactiveInstanceName}" \
                        --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
                    """, returnStdout: true).trim()

                    sh """
                        scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/devops-key.pem index.html ec2-user@${publicIp}:/tmp/index.html
                        ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/devops-key.pem ec2-user@${publicIp} '
                            sudo cp /tmp/index.html /var/www/html/index.html
                            sudo systemctl restart httpd
                        '
                    """
                    echo "Deployed to ${env.INACTIVE_ENV}"
                }
            }
        }
        stage('Perform Health Check') {
            steps {
                script {
                    def publicIp = sh(script: """
                        aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=${env.INACTIVE_ENV.toLowerCase()}" \
                        --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
                    """, returnStdout: true).trim()

                    def status = sh(script: "curl -f -s http://${publicIp} > /dev/null && echo 'OK' || echo 'FAIL'", returnStdout: true).trim()
                    if (status != 'OK') {
                        error "Health check failed on ${env.INACTIVE_ENV}"
                    }
                    echo "Health check PASSED"
                }
            }
        }
        stage('Switch ALB Traffic') {
            steps {
                script {
                    def newTGArn = sh(script: """
                        aws elbv2 describe-target-groups --names ${env.INACTIVE_TG} \
                        --query 'TargetGroups[0].TargetGroupArn' --output text
                    """, returnStdout: true).trim()

                    def listenerArn = sh(script: """
                        aws elbv2 describe-listeners \
                        --load-balancer-arn \$(aws elbv2 describe-load-balancers --names ${ALB_NAME} --query 'LoadBalancers[0].LoadBalancerArn' --output text) \
                        --query 'Listeners[0].ListenerArn' --output text
                    """, returnStdout: true).trim()

                    sh """
                        aws elbv2 modify-listener \
                        --listener-arn ${listenerArn} \
                        --default-actions Type=forward,TargetGroupArn=${newTGArn}
                    """
                    echo "Traffic switched to ${env.INACTIVE_ENV} ✅"
                }
            }
        }
    }
    post {
        failure {
            script {
                echo "Deployment failed - Rolling back..."
                def originalTGArn = sh(script: """
                    aws elbv2 describe-target-groups --names ${env.ACTIVE_TG} \
                    --query 'TargetGroups[0].TargetGroupArn' --output text
                """, returnStdout: true).trim()

                def listenerArn = sh(script: """
                    aws elbv2 describe-listeners \
                    --load-balancer-arn \$(aws elbv2 describe-load-balancers --names ${ALB_NAME} --query 'LoadBalancers[0].LoadBalancerArn' --output text) \
                    --query 'Listeners[0].ListenerArn' --output text
                """, returnStdout: true).trim()

                sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${listenerArn} \
                    --default-actions Type=forward,TargetGroupArn=${originalTGArn}
                """
                echo "Rollback completed - Traffic reverted"
            }
        }
    }
}