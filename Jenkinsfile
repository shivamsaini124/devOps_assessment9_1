
pipeline {
    agent any

    environment {
        IMAGE = 'YOUR_DOCKERHUB_USERNAME/devops-assessment9-1'
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

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE:$BUILD_NUMBER .
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -eu

                    docker run -d \
                      --name devops-assessment9-1-test \
                      -p 18080:80 \
                      $IMAGE:$BUILD_NUMBER

                    trap 'docker rm -f devops-assessment9-1-test >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 15); do
                      if curl -fsS http://127.0.0.1:18080 -o /dev/null; then
                        echo "Application test passed"
                        exit 0
                      fi
                      sleep 2
                    done

                    echo "Application test failed"
                    exit 1
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push $IMAGE:$BUILD_NUMBER
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KCFG'
                    )
                ]) {
                    script {
                        try {
                            sh '''
                                export KUBECONFIG="$KCFG"

                                # Replace the placeholder in the manifest.
                                sed "s|YOUR_DOCKERHUB_USERNAME|${IMAGE%%/*}|" \
                                  deployment.yaml > /tmp/devops-assessment9-1.yaml

                                kubectl apply -f /tmp/devops-assessment9-1.yaml

                                kubectl set image \
                                  deployment/$DEPLOYMENT \
                                  $CONTAINER=$IMAGE:$BUILD_NUMBER

                                kubectl rollout status \
                                  deployment/$DEPLOYMENT \
                                  --timeout=120s

                                kubectl get pods
                                kubectl get svc $SERVICE
                            '''
                        } catch (err) {
                            echo 'Deployment failed. Attempting rollback.'

                            sh '''
                                export KUBECONFIG="$KCFG"

                                kubectl rollout undo deployment/$DEPLOYMENT

                                kubectl rollout status \
                                  deployment/$DEPLOYMENT \
                                  --timeout=120s
                            '''

                            error('Deployment failed; rollback attempted.')
                        }
                    }
                }
            }
        }
    }
}
