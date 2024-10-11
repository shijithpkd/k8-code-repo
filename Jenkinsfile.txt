pipeline {
    agent any

    environment {
        // Define environment variables
        SONARQUBE_SCANNER_HOME = tool(name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation')
        SONARQUBE_URL = 'http://34.28.223.41:9000/'  // SonarQube URL

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
                                -Dsonar.projectKey=my-flask-app \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONARQUBE_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t my-flask-app .'
                sh 'docker tag my-flask-app ${DOCKER_BFLASK_IMAGE}'
            }
        }

        stage('Test') {
            steps {
                sh 'docker run my-flask-app python -m pytest app/tests/'
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "Waiting for Quality Gate result..."
                    timeout(time: 15, unit: 'MINUTES') {  // Increased timeout
                        def qualityGate = waitForQualityGate()

                        if (qualityGate.status != 'OK') {
                            echo "${qualityGate.status}"
                            error "Quality Gate failed: ${qualityGate.status}"
                        } else {
                            echo "${qualityGate.status}"
                            echo "SonarQube Quality Gates Passed"
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDS}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin docker.io"
                    sh 'docker push ${DOCKER_BFLASK_IMAGE}'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh "sed -i 's|image: .*|image: ${DOCKER_BFLASK_IMAGE}:latest|' deployment.yaml"
                    withCredentials([file(credentialsId: 'kubernetes-config-file', variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f deployment.yaml'
                        sh 'kubectl apply -f service.yaml'
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
