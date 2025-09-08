i// Declarative Pipeline for CI/CD of Spring PetClinic application
// This pipeline automates the build, test, Docker image creation, security scan,
// and deployment to an Azure Kubernetes Service (AKS) cluster.

pipeline {
    agent any

    environment {
        // --- Jenkins Credentials Configuration ---
        // These are placeholders for credentials configured in Jenkins.
        // Go to "Manage Jenkins" -> "Manage Credentials" to add them.
        // Ensure you have a 'secret text' for SONAR_TOKEN.
        // Add 'Username with password' credentials for the ACR login.
        // Add 'Secret file' for the KUBECONFIG.

        // SonarQube credentials
        SONAR_TOKEN = credentials('sonar-token')

        // Azure Container Registry (ACR) login credentials
        AZURE_ACR_USERNAME = credentials('acr-username-password').username
        AZURE_ACR_PASSWORD = credentials('acr-username-password').password

        // Kubernetes credentials (KUBECONFIG)
        KUBE_CONFIG = credentials('aks-kubeconfig')

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
                // Requires the SonarQube Scanner for Jenkins plugin.
                // Replace `sonarqube` with the name of your configured SonarQube server in Jenkins.
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2588:sonar -Dsonar.projectKey=petclinic -Dsonar.token=${SONAR_TOKEN}'
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
                    sh "docker login ${env.ACR_URL} -u ${env.ACR_USERNAME} -p ${env.ACR_PASSWORD}"
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
