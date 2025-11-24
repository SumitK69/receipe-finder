pipeline {
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
    image: lachlanevenson/k8s-kubectl:latest
    command: ['cat']
    tty: true
    env:
    - name: KUBECONFIG
      value: /kube/config
    volumeMounts:
    - name: kubeconfig-secret
      mountPath: /kube/config
      subPath: kubeconfig

  - name: dind
    image: docker:dind
    args: ["--storage-driver=overlay2", "--insecure-registry=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"]
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

  volumes:
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret
'''
        }
    }

    stages {

        stage('Install + Build Frontend') {
            // when {
            //     anyOf {
            //         changeset "src/**"
            //         changeset "package*.json"
            //         changeset "vite.config.js"
            //         changeset "index.html"
            //         changeset "tailwind.config.*"
            //         changeset "postcss.config.*"
            //         expression { return currentBuild.number == 1 }
            //     }
            // }
            steps {
                container('node') {
                    sh '''
                        npm install
                        npm run build
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            // when {
            //     anyOf {
            //         changeset "Dockerfile"
            //         changeset "docker-compose.yml"
            //         changeset "src/**"
            //         changeset "package*.json"
            //         changeset "dist/**"
            //         expression { return currentBuild.number == 1 }
            //     }
            // }
            steps {
                container('dind') {
                    sh '''
                        sleep 10
                        docker build -t receipe-nutrition-finder:latest .
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    sh '''
                        sonar-scanner \
                            -Dsonar.projectKey=receipe-nutrition-finder \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                            -Dsonar.token=sqp_220f805a7b6c80a326c0a631a64603c65edda39b
                    '''
                }
            }
        }

        stage('Login to Nexus Registry') {
            // when {
            //     anyOf {
            //         changeset "Dockerfile"
            //         changeset "src/**"
            //         changeset "package*.json"
            //         changeset "dist/**"
            //         expression { return currentBuild.number == 1 }
            //     }
            // }
            steps {
                container('dind') {
                    sh '''
                        docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 -u admin -p Changeme@2025
                    '''
                }
            }
        }

        stage('Push to Nexus') {
            // when {
            //     anyOf {
            //         changeset "Dockerfile"
            //         changeset "src/**"
            //         changeset "package*.json"
            //         changeset "dist/**"
            //         expression { return currentBuild.number == 1 }
            //     }
            // }
            steps {
                container('dind') {
                    sh '''
                        docker tag receipe-nutrition-finder:latest nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401102/receipe-nutrition-finder:v1
                        docker push nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401102/receipe-nutrition-finder:v1
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            // when {
            //     anyOf {
            //         changeset "k8s/**"
            //         changeset "Dockerfile"
            //         changeset "src/**"
            //         expression { return currentBuild.number == 1 }
            //     }
            // }
            steps {
                container('kubectl') {
                    sh '''
                        # Create or update the image pull secret in the cluster
                        kubectl create secret docker-registry nexus-secret \
                          --namespace=2401102 \
                          --docker-server=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 \
                          --docker-username=admin \
                          --docker-password=Changeme@2025 \
                          --docker-email=user@example.com \
                          --dry-run=client -o yaml | kubectl apply -f -

                        # Apply the deployment and service
                        kubectl apply -f k8s/deployment.yaml
                        
                        # Wait for pod to be created and get detailed events
                        echo "--- Waiting for pod to generate events ---"
                        sleep 15
                        
                        echo "--- Describing Pod for debugging ---"
                        POD_NAME=$(kubectl get pods -n 2401102 -l app=receipe-nutrition-finder -o jsonpath='{.items[0].metadata.name}' || echo "pod-not-found")
                        if [ "$POD_NAME" != "pod-not-found" ]; then
                          kubectl describe pod $POD_NAME -n 2401102
                        else
                          echo "Could not find a pod to describe."
                        fi
                        echo "--- End of Pod Description ---"

                        # Wait for the deployment to be successful
                        kubectl rollout status deployment/receipe-nutrition-finder-deployment -n 2401102
                    '''
                }
            }
        }
    }
}
