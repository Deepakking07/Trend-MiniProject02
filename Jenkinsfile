pipeline {
    agent any

    environment {
        DOCKER_IMAGE      = 'deepakk007/finalproj01-dev'
        AWS_DEFAULT_REGION = 'us-east-1'
        EKS_CLUSTER_NAME   = 'trend-app-cluster'
        NAMESPACE          = 'trend'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/YourUser/YourRepo.git'
                script {
                    env.GIT_COMMIT_SHA = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.BUILD_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHA}"
                    echo "Building image tag: ${env.BUILD_TAG}"
                }
            }
        }

        stage('Build React App') {
            steps {
                script {
                    sh """
                        cd react-trend-deployment
                        echo "Building React application using Docker..."

                        docker run --rm -v \$(pwd):/app -w /app node:22-alpine sh -c "
                            echo 'Installing dependencies...'
                            npm install
                            echo 'Building React application...'
                            npm run build
                            echo 'Build complete!'
                            ls -la dist/
                        "

                        echo "Verifying build output on host:"
                        ls -la dist/
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        cd react-trend-deployment
                        echo "Building Docker image..."
                        docker build -t ${DOCKER_IMAGE}:${env.BUILD_TAG} .
                        docker tag ${DOCKER_IMAGE}:${env.BUILD_TAG} ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_IMAGE}:${env.BUILD_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        sh """
                            export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
                            echo "Deploying to EKS cluster..."

                            aws configure list
                            aws sts get-caller-identity

                            aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --query 'cluster.resourcesVpcConfig'

                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}

                            echo "Testing kubectl..."
                            kubectl version --client
                            kubectl config view --minify

                            if timeout 60 kubectl get nodes --request-timeout=45s; then
                                echo "Cluster connectivity OK"

                                echo "Updating deployment with image: ${DOCKER_IMAGE}:${env.BUILD_TAG}"
                                kubectl set image deployment/trend-app trend-app=${DOCKER_IMAGE}:${env.BUILD_TAG} -n ${NAMESPACE} --request-timeout=30s

                                kubectl rollout status deployment/trend-app -n ${NAMESPACE} --timeout=180s
                                kubectl get pods -n ${NAMESPACE}
                                kubectl get svc -n ${NAMESPACE}
                            else
                                echo "Cluster connectivity failed"
                                exit 1
                            fi
                        """
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'aws-credentials', region: "${AWS_DEFAULT_REGION}")]) {
                        sh """
                            LOAD_BALANCER_URL=\$(kubectl get svc trend-app-service -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                            echo "LoadBalancer URL: http://\$LOAD_BALANCER_URL"

                            for i in {1..5}; do
                                echo "Health check attempt \$i..."
                                if curl -f --max-time 10 http://\$LOAD_BALANCER_URL; then
                                    echo "Application is healthy!"
                                    break
                                else
                                    echo "Attempt \$i failed, retrying in 30 seconds..."
                                    sleep 30
                                fi
                            done
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                sh """
                    docker rmi ${DOCKER_IMAGE}:${env.BUILD_TAG} || true
                    docker system prune -f || true
                """
            }
        }
        success {
            echo 'Pipeline succeeded! Application deployed successfully.'
        }
        failure {
            echo 'Pipeline failed! Check the logs for details.'
        }
    }
}
