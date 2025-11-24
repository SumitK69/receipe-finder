pipeline {
  agent { docker { image 'node:20-alpine' args '-u root:root' } }

  environment {
    IMAGE_NAME = "${IMAGE_NAME ?: 'recipe-finder'}"
    DOCKER_REGISTRY = "${DOCKER_REGISTRY ?: ''}"
    DOCKER_CREDENTIALS_ID = "${DOCKER_CREDENTIALS_ID ?: ''}"
    DO_PUSH = "${DOCKER_PUSH ?: 'false'}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Lint') {
      steps {
        // Fail the build if lint errors are present. Remove `|| true` to enforce strictly.
        sh 'npm run lint || true'
      }
    }

    stage('Build') {
      steps {
        sh 'npm run build'
      }
    }

    stage('Archive') {
      steps {
        archiveArtifacts artifacts: 'dist/**', fingerprint: true
      }
    }

    stage('Docker: Build & Push') {
      when {
        expression { return env.DO_PUSH == 'true' && env.DOCKER_REGISTRY }
      }
      steps {
        script {
          def shortCommit = (env.GIT_COMMIT ?: 'local').take(7)
          def tag = "${env.BUILD_NUMBER}-${shortCommit}"
          def fullImage = "${env.DOCKER_REGISTRY}/${env.IMAGE_NAME}:${tag}"

          sh "docker build -t ${fullImage} ."

          withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh "echo $DOCKER_PASS | docker login ${env.DOCKER_REGISTRY} -u $DOCKER_USER --password-stdin"
            sh "docker push ${fullImage}"
          }

          echo "Pushed image: ${fullImage}"
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }
    success {
      echo 'Build succeeded.'
    }
    failure {
      echo 'Build failed.'
    }
  }
}
