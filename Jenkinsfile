// // pipeline {
// //   agent {
// //     docker {
// //       image 'node:20-alpine'
// //       args '-u root:root'
// //     }
// //   }

// //   environment {
// //     IMAGE_NAME = "${IMAGE_NAME ?: 'recipe-finder'}"
// //     DOCKER_REGISTRY = "${DOCKER_REGISTRY ?: ''}"
// //     DOCKER_CREDENTIALS_ID = "${DOCKER_CREDENTIALS_ID ?: ''}"
// //     DO_PUSH = "${DOCKER_PUSH ?: 'false'}"
// //   }

// //   options {
// //     buildDiscarder(logRotator(numToKeepStr: '10'))
// //   }

// //   stages {
// //     stage('Checkout') {
// //       steps {
// //         checkout([
// //           $class: 'GitSCM',
// //           branches: [[name: '*/main']],
// //           userRemoteConfigs: [[url: 'https://github.com/SumitK69/receipe-nutrition-finder.git']]
// //         ])
// //       }
// //     }

// //     stage('Install') {
// //       steps {
// //         sh 'npm ci'
// //       }
// //     }

// //     stage('Lint') {
// //       steps {
// //         // Fail the build if lint errors are present. Remove `|| true` to enforce strictly.
// //         sh 'npm run lint || true'
// //       }
// //     }

// //     stage('Build') {
// //       steps {
// //         sh 'npm run build'
// //       }
// //     }

// //     stage('Archive') {
// //       steps {
// //         archiveArtifacts artifacts: 'dist/**', fingerprint: true
// //       }
// //     }
// //   }

// //   post {
// //     success {
// //       echo 'Build succeeded.'
// //     }
// //     failure {
// //       echo 'Build failed.'
// //     }
// //   }
// // }

// pipeline {
//     agent {
//         kubernetes {
//             yaml '''
// apiVersion: v1
// kind: Pod
// spec:
//   containers:

//   - name: node
//     image: node:18
//     command: ['cat']
//     tty: true

//   - name: sonar-scanner
//     image: sonarsource/sonar-scanner-cli
//     command: ['cat']
//     tty: true

//   - name: kubectl
//     image: lachlanevenson/k8s-kubectl:latest
//     command: ['cat']
//     tty: true
//     env:
//     - name: KUBECONFIG
//       value: /kube/config
//     volumeMounts:
//     - name: kubeconfig-secret
//       mountPath: /kube/config
//       subPath: kubeconfig

//   - name: dind
//     image: docker:dind
//     args: ["--storage-driver=overlay2", "--insecure-registry=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"]
//     securityContext:
//       privileged: true
//     env:
//     - name: DOCKER_TLS_CERTDIR
//       value: ""
//   volumes:
//   - name: kubeconfig-secret
//     secret:
//       secretName: kubeconfig-secret
// '''
//         }
//     }

//     stages {

//         stage('Install + Build Frontend') {
//             steps {
//                 container('node') {
//                     sh '''
//                         npm install
//                         npm run build
//                     '''
//                 }
//             }
//         }

//         stage('Build Docker Image') {
//             steps {
//                 container('dind') {
//                     sh '''
//                         sleep 10
//                         docker build -t recipe-finder:latest .
//                     '''
//                 }
//             }
//         }

//         stage('SonarQube Analysis') {
//             steps {
//                 container('sonar-scanner') {
//                     sh '''
//                         sonar-scanner \
//                             -Dsonar.projectKey=2401063-ashutosh \
//                             -Dsonar.sources=. \
//                             -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
//                             -Dsonar.login=sqp_fec0d2cd0d6849ed77e9d26ed8ae79e2a03b2844
//                     '''
//                 }
//             }
//         }


//         stage('Login to Nexus Registry') {
//             steps {
//                 container('dind') {
//                     sh '''
//                         docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 -u admin -p Changeme@2025
//                     '''
//                 }
//             }
//         }


//         stage('Push to Nexus') {
//             steps {
//                 container('dind') {
//                     sh '''
//                         docker tag recipe-finder:latest nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401102/recipe-finder:v1
//                         docker push nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401102/recipe-finder:v1
//                     '''
//                 }
//             }
//         }

//         stage('Deploy to Kubernetes') {
//             steps {
//                 container('kubectl') {
//                     sh '''
//                         set -x
//                         ls -la
//                         ls -la k8s
//                         kubectl version
//                         kubectl config view
//                         kubectl apply -f k8s/deployment.yaml -n 2401102
//                         kubectl apply -f k8s/service.yaml -n 2401102
//                         kubectl get all -n 2401102
//                         kubectl rollout status deployment/recipe-finder-deployment -n 2401102
//                     '''
//                 }
//             }
//         }
//     }
// }

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
            when {
                anyOf {
                    changeset "src/**"
                    changeset "package*.json"
                    changeset "vite.config.js"
                    changeset "index.html"
                    changeset "tailwind.config.*"
                    changeset "postcss.config.*"
                    expression { return currentBuild.number == 1 }
                }
            }
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
            when {
                anyOf {
                    changeset "Dockerfile"
                    changeset "src/**"
                    changeset "package*.json"
                    changeset "dist/**"
                    expression { return currentBuild.number == 1 }
                }
            }
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
            when {
                anyOf {
                    changeset "Dockerfile"
                    changeset "src/**"
                    changeset "package*.json"
                    changeset "dist/**"
                    expression { return currentBuild.number == 1 }
                }
            }
            steps {
                container('dind') {
                    sh '''
                        docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 -u admin -p Changeme@2025
                    '''
                }
            }
        }

        stage('Push to Nexus') {
            when {
                anyOf {
                    changeset "Dockerfile"
                    changeset "src/**"
                    changeset "package*.json"
                    changeset "dist/**"
                    expression { return currentBuild.number == 1 }
                }
            }
            steps {
                container('dind') {
                    sh '''
                        docker tag receipe-nutrition-finder:latest nexusx-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401102/receipe-nutrition-finder:v1
                        docker push nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401102/receipe-nutrition-finder:v1
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                anyOf {
                    changeset "k8s/**"
                    changeset "Dockerfile"
                    changeset "src/**"
                    expression { return currentBuild.number == 1 }
                }
            }
            steps {
                container('kubectl') {
                    sh '''
                        set -x
                        kubectl version
                        kubectl config view
                        kubectl apply -f k8s/deployment.yaml -n 2401102
                        kubectl apply -f k8s/service.yaml -n 2401102
                        kubectl get all -n 2401102
                        kubectl rollout status deployment/receipe-nutrition-finder-deployment -n 2401102
                    '''
                }
            }
        }
    }
}
