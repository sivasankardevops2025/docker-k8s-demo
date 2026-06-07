pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'devopsrk16'
        IMAGE_NAME = 'cicd-webapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_DEPLOYMENT = 'cicd-webapp-deployment'
        K8S_SERVICE = 'cicd-webapp-service'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Pulling latest code from GitHub'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image'
                sh 'docker build -t $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG .'
                sh 'docker tag $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG $DOCKERHUB_USERNAME/$IMAGE_NAME:latest'
            }
        }

        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker image to Docker Hub'
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG'
                    sh 'docker push $DOCKERHUB_USERNAME/$IMAGE_NAME:latest'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Updating Kubernetes manifest with Docker Hub username'
                sh 'sed -i "s#DOCKERHUB_USERNAME#$DOCKERHUB_USERNAME#g" deployment.yaml'
                echo 'Deploying application to Minikube Kubernetes'
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
                echo 'Restarting deployment to pull latest image'
                sh 'kubectl rollout restart deployment/$K8S_DEPLOYMENT'
                sh 'kubectl rollout status deployment/$K8S_DEPLOYMENT'
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'kubectl get deployment'
                sh 'kubectl get pods -o wide'
                sh 'kubectl get svc'
            }
        }

        stage('Start Port Forward for Azure Access') {
            steps {
                echo 'Starting port-forward in background for Azure public access'
                sh '''
                pkill -f "kubectl port-forward.*cicd-webapp-service" || true
                nohup kubectl port-forward --address 0.0.0.0 service/cicd-webapp-service 30082:80 > port-forward.log 2>&1 &
                sleep 3
                cat port-forward.log || true
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline Completed Successfully'
            echo 'Application URL: http://AZURE-VM-PUBLIC-IP:30082'
        }
        failure {
            echo 'CI/CD Pipeline Failed. Check Console Output.'
        }
    }
}
