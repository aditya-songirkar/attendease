pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'songirkar'          // <-- change this
        IMAGE_NAME     = "${DOCKERHUB_USER}/attendease"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        K8S_NAMESPACE  = 'default'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/2 * * * *')   // also checks GitHub every ~2 min (webhook optional)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Build & Test') {
            agent {
                docker { image 'node:18-alpine'; reuseNode true }
            }
            steps {
                sh 'npm ci'
                sh 'node --check server/server.js'   // syntax check
                sh 'npm test'                         // Jest unit tests
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .
                """
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                withCredentials([
                    string(credentialsId: 'mongodb-uri', variable: 'MONGODB_URI'),
                    string(credentialsId: 'jwt-secret',  variable: 'JWT_SECRET')
                ]) {
                    sh '''
                        kubectl config use-context minikube

                        # Create/update the Secret from Jenkins credentials (never stored in Git)
                        kubectl create secret generic attendease-secret \
                          --from-literal=MONGODB_URI="$MONGODB_URI" \
                          --from-literal=JWT_SECRET="$JWT_SECRET" \
                          --dry-run=client -o yaml | kubectl apply -f -

                        # Inject the freshly built image tag into the Deployment and apply
                        sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml | kubectl apply -f -
                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status deployment/attendease --timeout=180s
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    kubectl get pods -l app=attendease
                    kubectl get deployment attendease
                    kubectl get svc attendease-service

                    URL=$(minikube service attendease-service --url | head -n 1)
                    echo "App URL: $URL"
                    sleep 5
                    curl -fsS "$URL" > /dev/null && echo "Application is reachable"
                '''
            }
        }
    }

    post {
        success { echo "Deployed ${IMAGE_NAME}:${IMAGE_TAG} successfully" }
        failure { echo 'Pipeline failed - check the stage logs above' }
    }
}
