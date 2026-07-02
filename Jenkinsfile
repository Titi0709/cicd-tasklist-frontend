pipeline {
  agent any

  environment {
    NODE_ENV = 'test'
    DOCKER_IMAGE = 'thibaultlefay/tasklist-frontend:latest'
    SONAR_HOST_URL = 'http://localhost:9000'
    NVM_DIR = "${HOME}/.nvm"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Setup Node.js with NVM') {
      steps {
        sh '''
          # Install NVM if not present
          if [ ! -d "$NVM_DIR" ]; then
            echo "Installing NVM..."
            curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
          fi
          
          # Load NVM
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
          
          # Install Node.js 20
          nvm install 20
          nvm use 20
          
          # Verify installation
          node --version
          npm --version
        '''
      }
    }

    stage('Install dependencies') {
      steps {
        sh '''
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
          nvm use 20
          npm ci
        '''
      }
    }

    stage('Run tests') {
      steps {
        sh '''
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
          nvm use 20
          npm test -- --run --coverage || true
        '''
      }
    }

    stage('Build frontend') {
      steps {
        sh '''
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
          nvm use 20
          npm run build
        '''
      }
    }

    stage('Quality analysis with SonarQube') {
      steps {
        script {
          withCredentials([string(credentialsId: 'sonar-frontend-token', variable: 'SONAR_TOKEN')]) {
            sh '''
              export NVM_DIR="$HOME/.nvm"
              [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
              nvm use 20
              
              npx sonar-scanner \
                -Dsonar.projectKey=cicd-tasklist-frontend \
                -Dsonar.projectName=cicd-tasklist-frontend \
                -Dsonar.sources=src \
                -Dsonar.tests=src/__tests__ \
                -Dsonar.language=ts \
                -Dsonar.sourceEncoding=UTF-8 \
                -Dsonar.host.url=${SONAR_HOST_URL} \
                -Dsonar.login=${SONAR_TOKEN} \
                -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/** \
                -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info || true
            '''
          }
        }
      }
    }

    stage('Security scan with Trivy') {
      steps {
        sh '''
          # Check if trivy is installed, if not install via docker
          if ! command -v trivy &> /dev/null; then
            echo "Trivy not found, using Docker..."
            docker run --rm -v $(pwd):/root aquasec/trivy fs /root --exit-code 0 --format table || true
          else
            trivy fs --exit-code 0 --format table . || true
          fi
        '''
      }
    }

    stage('Build Docker image') {
      steps {
        sh 'docker build -t $DOCKER_IMAGE .'
      }
    }

    stage('Security scan Docker image with Trivy') {
      steps {
        sh '''
          if ! command -v trivy &> /dev/null; then
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 0 --format table $DOCKER_IMAGE || true
          else
            trivy image --exit-code 0 --format table $DOCKER_IMAGE || true
          fi
        '''
      }
    }

    stage('Publish Docker image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push $DOCKER_IMAGE
            docker logout
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/junit.xml,coverage/**', allowEmptyArchive: true
      junit testResults: 'reports/junit.xml', allowEmptyResults: true
    }
    success {
      echo ' Frontend Pipeline completed successfully!'
    }
    failure {
      echo ' Frontend Pipeline failed!'
    }
  }
}
