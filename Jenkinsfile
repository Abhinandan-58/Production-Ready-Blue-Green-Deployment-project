pipeline {
agent any

```
environment {
    ALB_NAME = 'My-Blue-Green-ALB'
    BLUE_TG  = 'TG-Blue'
    GREEN_TG = 'TG-Green'
    AWS_DEFAULT_REGION = 'us-east-1'
}

stages {

    stage('Pull Code from GitHub') {
        steps {
            git branch: 'main', url: 'https://github.com/Abhinandan-58/Production-Ready-Blue-Green-Deployment-project.git'
        }
    }

    stage('Determine Active Environment') {
        steps {
            script {

                def albArn = sh(script: """
                    aws elbv2 describe-load-balancers \
                    --names ${ALB_NAME} \
                    --query 'LoadBalancers[0].LoadBalancerArn' \
                    --output text
                """, returnStdout: true).trim()

                def listenerArn = sh(script: """
                    aws elbv2 describe-listeners \
                    --load-balancer-arn ${albArn} \
                    --query 'Listeners[0].ListenerArn' \
                    --output text
                """, returnStdout: true).trim()

                def currentTG = sh(script: """
                    aws elbv2 describe-listeners \
                    --listener-arn ${listenerArn} \
                    --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                    --output text
                """, returnStdout: true).trim()

                echo "Current Target Group ARN: ${currentTG}"

                if (currentTG.toLowerCase().contains('blue')) {
                    env.INACTIVE_ENV = 'green'
                    env.ACTIVE_TG = "${BLUE_TG}"
                    env.INACTIVE_TG = "${GREEN_TG}"
                } else {
                    env.INACTIVE_ENV = 'blue'
                    env.ACTIVE_TG = "${GREEN_TG}"
                    env.INACTIVE_TG = "${BLUE_TG}"
                }

                echo "Deploying to: ${env.INACTIVE_ENV}"
            }
        }
    }

    stage('Deploy to Inactive Server') {
        steps {
            script {

                def publicIp = sh(script: """
                    aws ec2 describe-instances \
                    --filters Name=tag:Name,Values=${env.INACTIVE_ENV} \
                    --query 'Reservations[0].Instances[0].PublicIpAddress' \
                    --output text
                """, returnStdout: true).trim()

                echo "Server IP: ${publicIp}"

                sh """
                    scp -o StrictHostKeyChecking=no \
                    -i /var/lib/jenkins/devops-key.pem \
                    index.html ec2-user@${publicIp}:/tmp/index.html

                    ssh -o StrictHostKeyChecking=no \
                    -i /var/lib/jenkins/devops-key.pem \
                    ec2-user@${publicIp} '
                    sudo cp /tmp/index.html /var/www/html/index.html
                    sudo systemctl restart httpd
                    '
                """

            }
        }
    }

    stage('Health Check') {
        steps {
            script {

                def publicIp = sh(script: """
                    aws ec2 describe-instances \
                    --filters Name=tag:Name,Values=${env.INACTIVE_ENV} \
                    --query 'Reservations[0].Instances[0].PublicIpAddress' \
                    --output text
                """, returnStdout: true).trim()

                def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${publicIp}", returnStdout: true).trim()

                if (status != "200") {
                    error "Health check failed"
                }

                echo "Health Check Passed"
            }
        }
    }

    stage('Switch Traffic') {
        steps {
            script {

                def newTGArn = sh(script: """
                    aws elbv2 describe-target-groups \
                    --names ${env.INACTIVE_TG} \
                    --query 'TargetGroups[0].TargetGroupArn' \
                    --output text
                """, returnStdout: true).trim()

                def albArn = sh(script: """
                    aws elbv2 describe-load-balancers \
                    --names ${ALB_NAME} \
                    --query 'LoadBalancers[0].LoadBalancerArn' \
                    --output text
                """, returnStdout: true).trim()

                def listenerArn = sh(script: """
                    aws elbv2 describe-listeners \
                    --load-balancer-arn ${albArn} \
                    --query 'Listeners[0].ListenerArn' \
                    --output text
                """, returnStdout: true).trim()

                sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${listenerArn} \
                    --default-actions Type=forward,TargetGroupArn=${newTGArn}
                """

                echo "Traffic switched to ${env.INACTIVE_ENV}"
            }
        }
    }
}

post {
    failure {
        echo "Deployment Failed"
    }

    success {
        echo "Blue-Green Deployment Successful"
    }
}
```

}
