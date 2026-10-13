pipeline {
    agent any

    parameters {
        string(
            name: 'BRANCH',
            defaultValue: 'main',
            description: 'Branch to build'
        )
        string(
            name: 'REPO_URL',
            defaultValue: 'https://github.com/Moaz-Elgendy/jenkins-k8s-lab.git',
            description: 'Git repository URL'
        )
        string(
            name: 'SCM_CRED_ID',
            defaultValue: 'jenkins-token',
            description: 'Jenkins credentials ID for the Git SCM checkout'
        )
        string(
            name: 'DOCKER_CRED_ID',
            defaultValue: 'docker',
            description: 'Jenkins credentials ID holding the Docker Hub username + PAT'
        )
        string(
            name: 'KUBECONFIG_PATH',
            defaultValue: '/var/jenkins_home/.kube/config',
            description: 'Path to the kubeconfig inside the Jenkins agent that points at minikube'
        )
    }

    environment {
        APP_NAME   = 'new-app-nti'
        DEPLOY_DIR = 'k8s'
        SERVICE    = 'new-app-nti-service'
        DEPLOYMENT = 'new-app-nti'
    }

    stages {
        stage('Getting Repo files') {
            steps {
                git branch: "${params.BRANCH}", credentialsId: "${params.SCM_CRED_ID}", url: "${params.REPO_URL}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build -t "$APP_NAME:$BUILD_NUMBER" .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${params.DOCKER_CRED_ID}",
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    withEnv([
                        "IMAGE=${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}"
                    ]) {
                        sh '''
                            set -eu
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                            docker tag "$APP_NAME:$BUILD_NUMBER" "$IMAGE"
                            docker push "$IMAGE"
                        '''
                    }
                }
            }
        }

        stage('Prepare Manifests') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${params.DOCKER_CRED_ID}",
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    withEnv([
                        "IMAGE=${DOCKER_USERNAME}/${APP_NAME}:${BUILD_NUMBER}"
                    ]) {
                        sh '''
                            set -eu
                            test -f "$DEPLOY_DIR/deployment.yaml" || { echo "missing $DEPLOY_DIR/deployment.yaml"; exit 1; }
                            test -f "$DEPLOY_DIR/service.yaml"     || { echo "missing $DEPLOY_DIR/service.yaml"; exit 1; }

                            sed "s|IMAGE_PLACEHOLDER|$IMAGE|" "$DEPLOY_DIR/deployment.yaml" > rendered-deployment.yaml

                            if grep -q IMAGE_PLACEHOLDER rendered-deployment.yaml; then
                                echo "placeholder was not substituted"; exit 1
                            fi
                            echo "rendered image:"
                            grep image: rendered-deployment.yaml
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withEnv(["KUBECONFIG=${params.KUBECONFIG_PATH}"]) {
                    sh '''
                        set -eu

                        command -v kubectl >/dev/null || { echo "kubectl not found on the agent"; exit 1; }
                        test -r "$KUBECONFIG" || { echo "kubeconfig not readable at $KUBECONFIG"; exit 1; }

                        kubectl config current-context
                        kubectl apply -f rendered-deployment.yaml
                        kubectl apply -f "$DEPLOY_DIR/service.yaml"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withEnv(["KUBECONFIG=${params.KUBECONFIG_PATH}"]) {
                    sh '''
                        set -eu

                        kubectl rollout status "deployment/$DEPLOYMENT" --timeout=180s

                        echo "--- pods ---"
                        kubectl get pods -l "app=$APP_NAME" -o wide

                        echo "--- services ---"
                        kubectl get svc "$SERVICE"

                        IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                        echo "node: $IP"
                        echo "app:   http://$IP:30080/"
                    '''
                }
            }
        }
    }

    post {
        failure {
            withEnv(["KUBECONFIG=${params.KUBECONFIG_PATH}"]) {
                sh '''
                    kubectl get pods -l "app=$APP_NAME" -o wide 2>/dev/null || true
                    kubectl describe deployment "$DEPLOYMENT" 2>/dev/null | tail -30 || true
                '''
            }
        }
    }
}