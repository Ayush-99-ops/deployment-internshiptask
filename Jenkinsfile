pipeline {
    agent any

    environment {
        EC2_SSH_CREDENTIALS_ID = "ec2-ssh-key"
        EC2_HOST               = "YOUR_EC2_PUBLIC_IP_OR_DNS"
        EC2_USER               = "ubuntu"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Images') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${EC2_SSH_CREDENTIALS_ID}",
                    keyFileVariable: 'EC2_KEY'
                )]) {
                    sh """
                        ssh -i $EC2_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            cd ~/deployment-task
                            git pull origin main

                            docker build -t vote:latest   ./vote
                            docker build -t result:latest ./result
                            docker build -t worker:latest ./worker

                            docker save vote:latest   | sudo k3s ctr images import -
                            docker save result:latest | sudo k3s ctr images import -
                            docker save worker:latest | sudo k3s ctr images import -
                        '
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${EC2_SSH_CREDENTIALS_ID}",
                    keyFileVariable: 'EC2_KEY'
                )]) {
                    sh """
                        ssh -i $EC2_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            kubectl apply -f ~/deployment-task/k8s/postgres-secret.yaml
                            kubectl apply -f ~/deployment-task/k8s/redis-deployment.yaml
                            kubectl wait --for=condition=ready pod -l app=redis --timeout=60s
                            kubectl apply -f ~/deployment-task/k8s/vote-deployment.yaml
                            kubectl apply -f ~/deployment-task/k8s/result-deployment.yaml
                            kubectl apply -f ~/deployment-task/k8s/worker-deployment.yaml
                            kubectl rollout status deployment/vote   --timeout=120s
                            kubectl rollout status deployment/result --timeout=120s
                            kubectl rollout status deployment/worker --timeout=120s
                        '
                    """
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${EC2_SSH_CREDENTIALS_ID}",
                    keyFileVariable: 'EC2_KEY'
                )]) {
                    sh """
                        ssh -i $EC2_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            curl -sf http://localhost:31000 -o /dev/null && echo "Vote app OK"   || echo "Vote app FAILED"
                            curl -sf http://localhost:31001 -o /dev/null && echo "Result app OK" || echo "Result app FAILED"
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful! Build #${env.BUILD_NUMBER}"
        }
        failure {
            echo "Pipeline failed. Check the logs above."
        }
    }
}
