pipeline {
    // 1. Kubernetes Agent Definition: Defines the pod with all necessary tools.
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:

  - name: node
    image: node:18
    command: ['cat']
    tty: true

  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command: ['cat']
    tty: true

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
    securityContext:
      runAsUser: 0
      readOnlyRootFilesystem: false
    env:
    - name: KUBECONFIG
      value: /kube/config
    volumeMounts:
    - name: kubeconfig-secret
      mountPath: /kube/config
      subPath: kubeconfig

  - name: dind
    image: docker:dind
    // Essential for DIND (Docker-in-Docker) service to function correctly
    args: ["--storage-driver=overlay2"]
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

  volumes:
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret // Ensure this Secret exists in your Jenkins namespace
'''
        }
    }

    environment {
        // --- CRITICAL FIX: Use internal cluster DNS for the agent pod ---
        SONAR_HOST = "http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
        SONAR_TOKEN = credentials('sonar-token-ecom-2401072')

        // Nexus Registry Details
        NEXUS_URL = "nexus.imcc.com" // Use domain only for docker login/tagging
        NEXUS_REPO = "repository/docker-hosted" // Repository path for pushing
        IMAGE_NAME = "html-css-js-project"
        K8S_NAMESPACE = "2401072-ecom" // Assuming a namespace based on your project key
        
        // Nexus Credential ID added for consistency
        NEXUS_CREDS_ID = 'nexus-creds-ecom-2401072' 
    }

    stages {

        stage('Clean Old Workspace Dockerfile') {
            steps {
                // Execute cleanup commands inside the 'node' container
                container('node') {
                    sh '''
                        echo "Deleting ALL existing Dockerfiles in Kubernetes workspace..."
                        find . -name "Dockerfile" -type f -print -delete || true
                        echo "Workspace cleaned."
                    '''
                }
            }
        }

        stage('Checkout Code') {
            // Jenkins default agent handles git checkout (no container needed)
            steps {
                git branch: 'main',
                url: 'https://github.com/sam160203/ecommerce'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Use the SonarQube scanner container
                container('sonar-scanner') {
                    withSonarQubeEnv('sonar-ecom-2401072') {
                        sh """
                            echo "Starting SonarQube analysis for ${IMAGE_NAME}..."
                            sonar-scanner \\
                            -Dsonar.projectKey=2401072_ecom \\ // FIX: Updated Project Key
                            -Dsonar.sources=. \\
                            -Dsonar.host.url=${SONAR_HOST}
                            // SONAR_TOKEN is implicitly picked up by withSonarQubeEnv and the sonar-scanner properties
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                // Execute Docker commands inside the 'dind' (Docker-in-Docker) container
                container('dind') {
                    sh '''
                        sleep 10 // Give DIND time to initialize
                        echo "Building Docker image: ${IMAGE_NAME}:latest"
                        docker build -t ${IMAGE_NAME}:latest .
                    '''
                }
            }
        }

        stage('Push to Nexus') {
            steps {
                // Execute Docker commands inside the 'dind' container, using Jenkins credentials
                container('dind') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: NEXUS_CREDS_ID,
                            usernameVariable: 'NEXUS_USER',
                            passwordVariable: 'NEXUS_PASS'
                        )
                    ]) {
                        sh """
                            echo "Logging in to Nexus Docker Registry: ${NEXUS_URL}"
                            docker login ${NEXUS_URL} -u $NEXUS_USER -p $NEXUS_PASS

                            IMAGE_TAG="${NEXUS_URL}/${NEXUS_REPO}/${IMAGE_NAME}:${BUILD_NUMBER}"
                            
                            echo "Tagging image: ${IMAGE_TAG}"
                            docker tag ${IMAGE_NAME}:latest ${IMAGE_TAG}

                            echo "Pushing image to Nexus..."
                            docker push ${IMAGE_TAG}

                            // Also push the 'latest' tag for convenience
                            docker tag ${IMAGE_NAME}:latest ${NEXUS_URL}/${NEXUS_REPO}/${IMAGE_NAME}:latest
                            docker push ${NEXUS_URL}/${NEXUS_REPO}/${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Create Namespace') {
            steps {
                // Execute kubectl commands inside the 'kubectl' container
                container('kubectl') {
                    sh '''
                        echo "Creating namespace: ${K8S_NAMESPACE}"
                        kubectl create namespace ${K8S_NAMESPACE} || echo "Namespace already exists or creation failed"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // Execute kubectl commands inside the 'kubectl' container
                container('kubectl') {
                    sh """
                        echo "Applying Kubernetes manifests in namespace ${K8S_NAMESPACE}..."
                        kubectl apply -f k8s/deployment.yaml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/service.yaml -n ${K8S_NAMESPACE}

                        echo "Waiting for deployment rollout..."
                        kubectl rollout status deployment/${IMAGE_NAME}-deployment -n ${K8S_NAMESPACE} --timeout=120s || true
                        
                        kubectl get all -n ${K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('Debug Pods') {
            steps {
                // Execute debug commands inside the 'kubectl' container
                container('kubectl') {
                    sh """
                        echo "[DEBUG] Listing Pods and showing logs in ${K8S_NAMESPACE}..."
                        kubectl get pods -n ${K8S_NAMESPACE}
                        
                        // Get logs from the first pod in the namespace
                        POD_NAME=\$(kubectl get pods -n ${K8S_NAMESPACE} -o jsonpath='{.items[0].metadata.name}')
                        if [ ! -z "\$POD_NAME" ]; then
                            echo "--- Logs for pod: \$POD_NAME ---"
                            kubectl logs \$POD_NAME -n ${K8S_NAMESPACE} --tail=100 || true
                        else
                            echo "No pods found to debug."
                        fi
                    """
                }
            }
        }
    }
}