pipeline {
    agent any

    environment {
        // --- Other Environment Variables ---
        REPO_NAME = 'spring-petclinic'
        ACR_URL = 'petclinicregistry12345.azurecr.io'
        IMAGE_NAME = "${env.ACR_URL}/${REPO_NAME}:${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo '1. Cloning source code...'
                git branch: 'main', url: 'https://github.com/vanambharat/spring-petclinic.git'
            }
        }

        stage('Build & Unit Test') {
            steps {
                echo '2. Building with Maven and running unit tests...'
                // The `-B` flag runs in non-interactive batch mode.
                sh 'mvn clean package -B'
            }
        }

        stage('Sonar Analysis') {
            steps {
                echo '3. Running SonarQube analysis...'
                // The SonarQube Scanner for Jenkins plugin will provide the SonarQube server details.
                // It will automatically use the `sonar-token` credential you configured.
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2588:sonar -Dsonar.projectKey=petclinic -Dsonar.token=${env.SONAR_TOKEN}'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                echo '4. Creating Docker image and pushing to ACR...'
                // Build the Docker image
                sh "docker build -t ${env.IMAGE_NAME} ."
                
                // Login to ACR using the credentials from Jenkins
                withCredentials([usernamePassword(credentialsId: 'acr-username-password', passwordVariable: 'ACR_PASSWORD', usernameVariable: 'ACR_USERNAME')]) {
                    sh "docker login ${env.ACR_URL} -u ${env.ACR_USERNAME} -p ${ACR_PASSWORD}"
                }
                
                // Push the Docker image to ACR
                sh "docker push ${env.IMAGE_NAME}"
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo '5. Scanning Docker image for vulnerabilities with Trivy...'
                // Install trivy on the Jenkins agent first, then run this.
                sh "trivy image ${env.IMAGE_NAME}"
            }
        }

        stage('Deploy to AKS') {
            steps {
                echo '6. Deploying to Azure Kubernetes Service (AKS)...'
                withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBE_CONFIG_FILE')]) {
                    sh "export KUBECONFIG=${KUBE_CONFIG_FILE}"
                    sh "kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml"
                    sh "kubectl rollout status deployment/petclinic-deployment"
                }
            }
        }
    }
}
