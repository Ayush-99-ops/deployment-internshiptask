pipeline {
    agent any

    environment {
        // ─── Change these to your Docker Hub username and image names ───
        DOCKER_HUB_USER = "YOUR_DOCKERHUB_USERNAME"
        VOTE_IMAGE      = "${DOCKER_HUB_USER}/vote"
        RESULT_IMAGE    = "${DOCKER_HUB_USER}/result"
        WORKER_IMAGE    = "${DOCKER_HUB_USER}/worker"

        // Jenkins credential ID for Docker Hub (username + password)
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"

        // AWS EC2 SSH credential ID (PEM key stored in Jenkins)
        EC2_SSH_CREDENTIALS_ID = "ec2-ssh-key"

        // Public IP / hostname of your EC2 instance
        EC2_HOST = "YOUR_EC2_PUBLIC_IP_OR_DNS"
        EC2_USER = "ubuntu"
    }

    stages {

        // ── Stage 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ── Stage 2: Build Docker Images ──────────────────────────────────
        stage('Build Images') {
            parallel {
                stage('Build Vote') {
                    steps {
                        script {
                            def tag = "${env.BUILD_NUMBER}"
                            sh "docker build -t ${VOTE_IMAGE}:${tag} -t ${VOTE_IMAGE}:latest ./vote"
                        }
                    }
                }
                stage('Build Result') {
                    steps {
                        script {
                            def tag = "${env.BUILD_NUMBER}"
                            sh "docker build -t ${RESULT_IMAGE}:${tag} -t ${RESULT_IMAGE}:latest ./result"
                        }
                    }
                }
                stage('Build Worker') {
                    steps {
                        script {
                            def tag = "${env.BUILD_NUMBER}"
                            sh "docker build -t ${WORKER_IMAGE}:${tag} -t ${WORKER_IMAGE}:latest ./worker"
                        }
                    }
                }
            }
        }

        // ── Stage 3: Push to Docker Hub ───────────────────────────────────
        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    script {
                        def tag = "${env.BUILD_NUMBER}"
                        sh """
                            docker push ${VOTE_IMAGE}:${tag}
                            docker push ${VOTE_IMAGE}:latest
                            docker push ${RESULT_IMAGE}:${tag}
                            docker push ${RESULT_IMAGE}:latest
                            docker push ${WORKER_IMAGE}:${tag}
                            docker push ${WORKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        // ── Stage 4: Deploy to EC2 via kubectl ────────────────────────────
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${EC2_SSH_CREDENTIALS_ID}",
                    keyFileVariable: 'EC2_KEY'
                )]) {
                    script {
                        def tag = "${env.BUILD_NUMBER}"
                        sh """
                            scp -i $EC2_KEY -o StrictHostKeyChecking=no -r k8s/ ${EC2_USER}@${EC2_HOST}:/home/${EC2_USER}/k8s/

                            ssh -i $EC2_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                # Apply Neon DB secret and Redis (no postgres pod - using Neon DB)
                                kubectl apply -f /home/${EC2_USER}/k8s/postgres-secret.yaml
                                kubectl apply -f /home/${EC2_USER}/k8s/redis-deployment.yaml

                                # Wait for Redis to be ready
                                kubectl wait --for=condition=ready pod -l app=redis --timeout=60s

                                # Update image tags to current build then apply
                                sed -i "s|YOUR_DOCKERHUB_USERNAME/vote:latest|${VOTE_IMAGE}:${tag}|g"   /home/${EC2_USER}/k8s/vote-deployment.yaml
                                sed -i "s|YOUR_DOCKERHUB_USERNAME/result:latest|${RESULT_IMAGE}:${tag}|g" /home/${EC2_USER}/k8s/result-deployment.yaml
                                sed -i "s|YOUR_DOCKERHUB_USERNAME/worker:latest|${WORKER_IMAGE}:${tag}|g" /home/${EC2_USER}/k8s/worker-deployment.yaml

                                kubectl apply -f /home/${EC2_USER}/k8s/vote-deployment.yaml
                                kubectl apply -f /home/${EC2_USER}/k8s/result-deployment.yaml
                                kubectl apply -f /home/${EC2_USER}/k8s/worker-deployment.yaml

                                # Wait for rollout
                                kubectl rollout status deployment/vote   --timeout=120s
                                kubectl rollout status deployment/result --timeout=120s
                                kubectl rollout status deployment/worker --timeout=120s
                            '
                        """
                    }
                }
            }
        }

        // ── Stage 5: Smoke Test ───────────────────────────────────────────
        stage('Smoke Test') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "${EC2_SSH_CREDENTIALS_ID}",
                    keyFileVariable: 'EC2_KEY'
                )]) {
                    sh """
                        ssh -i $EC2_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            # Vote app should return HTTP 200
                            curl -sf http://localhost:31000 -o /dev/null && echo "Vote app OK" || echo "Vote app FAILED"
                            # Result app should return HTTP 200
                            curl -sf http://localhost:31001 -o /dev/null && echo "Result app OK" || echo "Result app FAILED"
                        '
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean up local Docker images to save disk space on Jenkins agent
            sh """
                docker rmi ${VOTE_IMAGE}:${env.BUILD_NUMBER}   || true
                docker rmi ${RESULT_IMAGE}:${env.BUILD_NUMBER} || true
                docker rmi ${WORKER_IMAGE}:${env.BUILD_NUMBER} || true
            """
        }
        success {
            echo "✅ Deployment successful! Build #${env.BUILD_NUMBER}"
        }
        failure {
            echo "❌ Pipeline failed. Check the logs above."
        }
    }
}
