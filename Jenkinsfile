
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'shivam3294/devops-assessment9-1'
        DEPLOYMENT = 'devops-assessment9-1'
        CONTAINER = 'devops-assessment9-1'
        SERVICE = 'devops-assessment9-1-service'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build --pull=false \
                        -t "$DOCKER_IMAGE:$BUILD_NUMBER" .
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -eu

                    NAME=devops-assessment9-1-test
                    PORT=18080

                    docker rm -f "$NAME" >/dev/null 2>&1 || true
                    trap 'docker rm -f "$NAME" >/dev/null 2>&1 || true' EXIT

                    docker run -d --name "$NAME" \
                        -p "$PORT:80" "$DOCKER_IMAGE:$BUILD_NUMBER"

                    for i in $(seq 1 15); do
                        if curl -fsS "http://127.0.0.1:$PORT/" -o /dev/null; then
                            echo "Application test passed"
                            exit 0
                        fi
                        sleep 2
                    done

                    echo "Application test failed"
                    docker logs "$NAME"
                    exit 1
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                            -u "$DOCKER_USER" --password-stdin

                        docker push "$DOCKER_IMAGE:$BUILD_NUMBER"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KCFG'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KCFG"

                        echo "Checking Kubernetes cluster..."
                        kubectl config current-context

                        echo "Applying Kubernetes configuration..."
                        sed "s|shivam3294/devops-assessment9-1:v1|$DOCKER_IMAGE:$BUILD_NUMBER|g" \
                            deployment.yaml > deployment-ci.yaml

                        kubectl apply -f deployment-ci.yaml

                        echo "Waiting for deployment..."
                        if ! kubectl rollout status \
                            deployment/"$DEPLOYMENT" --timeout=120s; then

                            echo "Rollout failed; attempting rollback..."
                            kubectl rollout undo deployment/"$DEPLOYMENT" || true
                            exit 1
                        fi

                        echo "Deployment status:"
                        kubectl get deployment "$DEPLOYMENT"

                        echo "Pod status:"
                        kubectl get pods -l app="$DEPLOYMENT"

                        echo "Service status:"
                        kubectl get service "$SERVICE"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '========================================'
            echo 'PLACEMENT PORTAL DEPLOYMENT SUCCESSFUL'
            echo '========================================'
            echo "Docker Image: ${DOCKER_IMAGE}:${BUILD_NUMBER}"
            echo 'Expected replicas: 2'
            echo 'Expected NodePort: 30080'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'PIPELINE FAILED - CHECK CONSOLE OUTPUT'
            echo '========================================'
        }

        always {
            sh 'rm -f deployment-ci.yaml || true'
        }
    }
}
