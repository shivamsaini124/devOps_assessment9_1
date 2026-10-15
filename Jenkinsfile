pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "shivam3294/devOps_assessment9_1"
    }
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/shivamsaini124/devOps_assessment8.git'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build --pull=false -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )
                ]) {
                    sh 'echo "$PASS" | docker login -u "$USER" --password-stdin'
                    sh 'docker push $DOCKER_IMAGE:latest'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG"

                        echo "Checking Kubernetes cluster..."
                        kubectl config current-context

                        echo "Applying Kubernetes configuration..."
                        kubectl apply -f deployment.yaml

                        echo "Waiting for deployment..."
                        kubectl rollout status deployment/devOps_assessment9_1

                        echo "Deployment status:"
                        kubectl get deployment devOps_assessment9_1

                        echo "Pod status:"
                        kubectl get pods -l app=devOps_assessment9_1

                        echo "Service status:"
                        kubectl get service devOps_assessment9_1-service
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo 'PROFILE DEPLOYMENT SUCCESSFUL'
            echo '========================================'
            echo "Docker Image: ${DOCKER_IMAGE}:latest"
            echo 'Replicas: 2'
            echo 'NodePort: 30080'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'PIPELINE FAILED'
            echo '========================================'
        }
    }
}