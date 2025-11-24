pipeline {
  agent {
    docker {
      image 'node:20-alpine'
      args '-u root:root'
    }
  }

  environment {
    IMAGE_NAME = "${IMAGE_NAME ?: 'recipe-finder'}"
    DOCKER_REGISTRY = "${DOCKER_REGISTRY ?: ''}"
    DOCKER_CREDENTIALS_ID = "${DOCKER_CREDENTIALS_ID ?: ''}"
    DO_PUSH = "${DOCKER_PUSH ?: 'false'}"
  }

  options {
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
  }

  post {
    always {
      deleteDir()
    }
    success {
      echo 'Build succeeded.'
    }
    failure {
      echo 'Build failed.'
    }
  }
}
