pipeline {
  agent any

  environment {
    NODE_ENV = 'test'
  }

  stages {
    stage('Install dependencies') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Run unit tests') {
      steps {
        sh 'npm test'
      }
    }

    stage('Build frontend') {
      steps {
        sh 'npm run build'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/junit.xml', allowEmptyArchive: true
    }
  }
}
