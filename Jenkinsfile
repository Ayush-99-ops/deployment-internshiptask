pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Images') {
            steps {
                sh """
                    docker build -t vote:latest   ./vote
                    docker build -t result:latest ./result
                    docker build -t worker:latest ./worker
                """
            }
        }

        stage('Import into K3s') {
            steps {
                sh """
                    docker save vote:latest   | sudo k3s ctr images import -
                    docker save result:latest | sudo k3s ctr images import -
                    docker save worker:latest | sudo k3s ctr images import -
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl apply -f k8s/redis-deployment.yaml
                    kubectl wait --for=condition=ready pod -l app=redis --timeout=60s
                    kubectl apply -f k8s/vote-deployment.yaml
                    kubectl apply -f k8s/result-deployment.yaml
                    kubectl apply -f k8s/worker-deployment.yaml
                    kubectl rollout status deployment/vote   --timeout=120s
                    kubectl rollout status deployment/result --timeout=120s
                    kubectl rollout status deployment/worker --timeout=120s
                """
            }
        }

        stage('Smoke Test') {
            steps {
                sh """
                    curl -sf http://localhost:31000 -o /dev/null && echo "Vote app OK"   || echo "Vote app FAILED"
                    curl -sf http://localhost:31001 -o /dev/null && echo "Result app OK" || echo "Result app FAILED"
                """
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
