pipeline {
    agent any

    environment {
        SONAR_HOST = "http://sonarqube.imcc.com/"
        SONAR_TOKEN = credentials('sonar-token-ecom-2401072')

        NEXUS_URL = "http://nexus.imcc.com/"
        IMAGE_NAME = "html-css-js-project"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/sam160203/ecommerce'
            }
        }

        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonar-ecom-2401072') {
            sh """
                sonar-scanner \
                -Dsonar.projectKey=2401072_ecom \
                -Dsonar.sources=. \
                -Dsonar.host.url=${SONAR_HOST}
            """
        }
    }
}


        stage('Build Docker Image') {
            steps { 
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Push to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-creds-ecom-2401072',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    sh """
                        docker login ${NEXUS_URL} -u $NEXUS_USER -p $NEXUS_PASS
                        docker tag ${IMAGE_NAME}:latest nexus.imcc.com/repository/docker-hosted/${IMAGE_NAME}:latest
                        docker push nexus.imcc.com/repository/docker-hosted/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                """
            }
        }
    }
}
