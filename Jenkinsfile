pipeline {
  agent any

  environment {
    NODE_ENV = 'test'
    DOCKER_IMAGE = 'your-dockerhub-thibaultlefay/tasklist-frontend:latest'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install dependencies') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Run tests') {
      steps {
        sh 'npm test'
      }
    }

    stage('Build frontend') {
      steps {
        sh 'npm run build'
      }
    }

    stage('Quality analysis') {
      steps {
        sh 'npx sonar-scanner -Dsonar.projectKey=tasklist-frontend -Dsonar.sources=src -Dsonar.host.url=http://localhost:9000 -Dsonar.login=$SONAR_TOKEN || true'
      }
    }

    stage('Security scan') {
      steps {
        sh 'npx trivy fs --exit-code 0 --format table . || true'
      }
    }

    stage('Build Docker image') {
      steps {
        sh 'docker build -t $DOCKER_IMAGE .' 
      }
    }

    stage('Publish Docker image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
          sh 'docker push $DOCKER_IMAGE'
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/junit.xml,coverage/**', allowEmptyArchive: true
    }
  }
}
