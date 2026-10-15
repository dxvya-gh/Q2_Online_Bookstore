pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "warmcoots/bookstore-app"
        STABLE_IMAGE = "warmcoots/bookstore-app:build-3"
        APP_NAME = "bookstore-app"
        IMAGE_TAG = "build-${BUILD_NUMBER}"
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/dxvya-gh/Q2_Online_Bookstore.git'
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -e
                    test -s index.html
                    grep -q "Campus Bookstore" index.html
                    grep -q "Clean Code" index.html
                    grep -q "Introduction to Algorithms" index.html
                    grep -q "A Brief History of Time" index.html
                    grep -q "1984" index.html
                    grep -q "Atomic Habits" index.html
                    grep -q "The Hobbit" index.html
                    echo "PASS: Bookstore HTML contains all six required books."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$IMAGE_TAG .'
            }
        }

        stage('Test Docker Image') {
            steps {
                sh 'docker image inspect $DOCKER_IMAGE:$IMAGE_TAG'
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
                    sh '''
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push "$DOCKER_IMAGE:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        withCredentials([
                            file(
                                credentialsId: 'kuberconfig',
                                variable: 'KUBECONFIG'
                            )
                        ]) {
                            sh '''
                                set -e
                                if [ "$SIMULATE_FAILURE" = "true" ]; then
                                    TARGET_IMAGE="$DOCKER_IMAGE:rollback-test-invalid"
                                else
                                    TARGET_IMAGE="$DOCKER_IMAGE:$IMAGE_TAG"
                                fi

                                kubectl --kubeconfig="$KUBECONFIG" set image \
                                  deployment/$APP_NAME \
                                  $APP_NAME=$TARGET_IMAGE

                                kubectl --kubeconfig="$KUBECONFIG" rollout status \
                                  deployment/$APP_NAME --timeout=90s
                            '''
                        }
                    } catch (Exception e) {
                        echo "Deployment failed. Restoring the known-good stable image."

                        withCredentials([
                            file(
                                credentialsId: 'kuberconfig',
                                variable: 'KUBECONFIG'
                            )
                        ]) {
                            sh '''
                                set -e
                                kubectl --kubeconfig="$KUBECONFIG" set image \
                                  deployment/$APP_NAME \
                                  $APP_NAME=$STABLE_IMAGE

                                kubectl --kubeconfig="$KUBECONFIG" rollout status \
                                  deployment/$APP_NAME --timeout=120s
                            '''
                        }

                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Bookstore CI/CD pipeline finished."
        }
    }
}
