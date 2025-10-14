pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        BACKEND_IMAGE = "1006sneha/backend-app:latest"
        FRONTEND_IMAGE = "1006sneha/frontend-app:latest"

        AWS_REGION = "us-east-1"
        EKS_CLUSTER = "app-eks-cluster"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/sneha1006lakade/placement-project.git'
            }
        }
        
        stage('Maven Build & Test') {
            steps {
                sh '''cd spring-backend
                      rm -rf target
                      mvn clean install -DskipTests  '''
            }
        }
        
        stage('test') {
            steps {
               withSonarQubeEnv('sonarqube') { 
                    sh '''cd spring-backend
                       mvn sonar:sonar -Dsonar.projectKey=placement-project  -Dsonar.projectName=placement-project -Dsonar.login=${SONAR_AUTH_TOKEN}\'\'
               '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
              }
            }
        }

        stage('Build Backend Docker Image') {
            steps {
                dir('spring-backend') {
                    sh "docker build -t ${BACKEND_IMAGE} ."
                }
            }
        }

        stage('Build Frontend Docker Image') {
            steps {
                dir('angular-frontend') {
                    sh "docker build -t ${FRONTEND_IMAGE} ."
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
            }
        }

        stage('Push Docker Images') {
            steps {
                sh "docker push ${BACKEND_IMAGE}"
                sh "docker push ${FRONTEND_IMAGE}"
            }
        }

        stage('Apply Kubernetes Configs & Secrets') {
            steps {
                 withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                  sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

                    kubectl apply -f k8s/backend-secret.yaml
                    kubectl apply -f k8s/frontend-configmap.yaml
                 """
                 }
            }
        }

        stage('Deploy Backend to EKS') {
            steps {
                 withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
                  sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

                    kubectl get secret regcred || kubectl create secret docker-registry regcred \
                       --docker-server=https://index.docker.io/v1/ \
                       --docker-username=$DOCKERHUB_CREDENTIALS_USR \
                       --docker-password=$DOCKERHUB_CREDENTIALS_PSW
                    
                    kubectl apply -f k8s/backend-deployment.yaml
                    kubectl apply -f k8s/backend-service.yaml
                """
                }  
            }
        }

        stage('Deploy Frontend to EKS') {
            steps {
                 withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
                  sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

                    kubectl get secret regcred || kubectl create secret docker-registry regcred \
                       --docker-server=https://index.docker.io/v1/ \
                       --docker-username=$DOCKERHUB_CREDENTIALS_USR \
                       --docker-password=$DOCKERHUB_CREDENTIALS_PSW

                    kubectl apply -f k8s/frontend-deployment.yaml
                    kubectl apply -f k8s/frontend-service.yaml
                    kubectl apply -f k8s/ingress.yaml
                """
                }  
            }
        }
    }

    post {
        success {
              mail to: 'team@example.com',
                  subject: "Build Successful",
                  body: "Your build completed successfully."
        }
        failure {
              mail to: 'team@example.com',
                  subject: "Build Failed",
                 body: "Build failed. Please check logs."
        }
    }
}
