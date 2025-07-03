pipeline {
    agent none

    environment {
        SERVICE_NAME = "frontend"
        BRANCH = "${env.BRANCH_NAME}"
        DOCKER_IMAGE = "pdock855/dr-front:${SERVICE_NAME}-${env.BRANCH_NAME}-${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            agent { label 'built-in' }  
            steps {
                git branch: BRANCH, credentialsId: 'github', url: 'https://github.com/pawanr-98/frontend-t.git'
            }
        }

        stage('Frontend Build and Push') {
            agent { label 'built-in' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker build -t $DOCKER_IMAGE .
                    docker push $DOCKER_IMAGE
                    docker logout
                    """
                }
            }
        }

        stage('Frontend Deployment') {
            agent { label 'master-node' }
            steps {
                script {
                    def namespace = ''
                    if (env.BRANCH_NAME == 'main') {
                        namespace = 'ums-prod'
                    } else if (env.BRANCH_NAME == 'uat') {
                        namespace = 'ums-uat'
                    } else {
                        error("❌ Unsupported branch: ${env.BRANCH_NAME}. Only 'main' and 'uat' are allowed.")
                    }

                    withCredentials([string(credentialsId: 'ums-cluster', variable: 'CLUSTER_CONTEXT')]) {
                        sh """
                        kubectl config use-context $CLUSTER_CONTEXT
                        kubectl apply -f k8s-manifests/deployment.yaml -n ${namespace}
                        kubectl apply -f k8s-manifests/service.yaml -n ${namespace}
                        kubectl apply -f k8s-manifests/ingress.yaml -n ${namespace}
                        kubectl set image deployment/ums-frontend ums-frontend=$DOCKER_IMAGE -n ${namespace}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployed $SERVICE_NAME to namespace: ${env.BRANCH_NAME}"
        }
        failure {
            echo "Deployment failed for $SERVICE_NAME to ${env.BRANCH_NAME}"
        }
    }
}
