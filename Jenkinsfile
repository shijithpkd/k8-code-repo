pipeline {
    agent any

    environment {
        // Define environment variables
        SONARQUBE_SCANNER_HOME = tool(name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation')
        SONARQUBE_URL = 'http://34.67.22.183:9000/'  // SonarQube URL
        DOCKER_REGISTRY = 'docker.io'  // Docker registry URL
       // DOCKER_IMAGE_NAME = 'my-nginx-image'  // Replace with your Docker image name
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Using Jenkins build number as the image tag
        HELM_RELEASE_NAME = 'nginx-release'  // Helm release name
        HELM_CHART_PATH = './nginx-chat'  // Path to your Helm chart
        KUBE_NAMESPACE = 'nginx'  // Kubernetes namespace for deployment
    }

    stages {
        stage('Checkout') {
            steps {
                // Checkout your source code from SCM
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Wrap the SonarQube analysis within the SonarQube environment.
                withSonarQubeEnv('SonarQube') {  // 'SonarQube' is the name of your SonarQube server in Jenkins
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONARQUBE_TOKEN')]) {
                        // Execute SonarQube Scanner
                        sh """
                            ${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=Nginx-Helm \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONARQUBE_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image with the specified tag
                    sh "docker build -t ${DOCKER_NGINX_IMAGE}:${IMAGE_TAG} Docker-files/."
                    
                    // Tag the image for the Docker registry
                    //sh "docker tag ${DOCKER_NGINX_IMAGE}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${DOCKER_NGINX_IMAGE}:${IMAGE_TAG}"
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    // Define variables for container name and test port
                    def containerName = "nginx-test-${env.BUILD_NUMBER}"
                    def testPort = 8080  // You can choose any available port

                    // Step 1: Run the Nginx container in detached mode
                    sh """
                        docker run -d --name ${containerName} -p ${testPort}:80 ${DOCKER_NGINX_IMAGE}:${IMAGE_TAG}
                    """

                    // Step 2: Wait for Nginx to start
                    sh 'sleep 15'  // Adjust the sleep duration if necessary

                    // Step 3: Perform health check using curl
                    // You can customize the URL and expected response based on your Nginx configuration
                    def response = sh(
                        script: "curl -s -o /dev/null -w \"%{http_code}\" http://localhost:${testPort}",
                        returnStdout: true
                    ).trim()

                    // Step 4: Validate the response
                    if (response != '200') {
                        error "Nginx server health check failed. Expected HTTP 200 but received ${response}."
                    } else {
                        echo "Nginx server is up and responding with HTTP 200."
                    }

                    // Step 5: Stop and remove the Nginx container
                    sh """
                        docker stop ${containerName}
                        docker rm ${containerName}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "Waiting for Quality Gate result..."
                    timeout(time: 15, unit: 'MINUTES') {  // Increased timeout
                        def qualityGate = waitForQualityGate()

                        if (qualityGate.status != 'OK') {
                            echo "Quality Gate Status: ${qualityGate.status}"
                            error "Quality Gate failed: ${qualityGate.status}"
                        } else {
                            echo "Quality Gate Status: ${qualityGate.status}"
                            echo "SonarQube Quality Gates Passed"
                        }
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDS}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    script {
                        // Log in to Docker registry
                        sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin ${DOCKER_REGISTRY}"
                        
                        // Push the Docker image to the registry
                        sh "docker push ${DOCKER_NGINX_IMAGE}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    // Trigger manual input for deployment
                    def userInput = input(
                        id: 'ProceedDeployment', message: 'Deploy to Kubernetes?', parameters: [
                            [$class: 'BooleanParameterDefinition', defaultValue: true, description: 'Proceed with deploying to Kubernetes?', name: 'Deploy']
                        ]
                    )

                    if (userInput) {
                        withCredentials([file(credentialsId: 'kubernetes-config-file', variable: 'KUBECONFIG')]) {
                            // (Optional) Validate Helm chart
                            sh "helm lint ${HELM_CHART_PATH}"

                            // Initialize Helm (optional, if needed)
                            sh 'helm version'

                            // Deploy using Helm, passing the image repository and tag as variables
                            sh """
                                helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} \
                                    --set image.repository=${DOCKER_NGINX_IMAGE} \
                                    --set image.tag=${IMAGE_TAG} \
                                    --namespace ${KUBE_NAMESPACE} \
                                    --create-namespace
                            """
                        }
                    } else {
                        echo "Deployment to Kubernetes was aborted by the user."
                        // Optionally, you can mark the build as unstable or failed
                        // currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }

    post {
        always {
            // Logout from Docker registry
            sh 'docker logout'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
