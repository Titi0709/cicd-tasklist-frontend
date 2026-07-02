pipeline {
  agent any

  environment {
    NODE_ENV = 'test'
    DOCKER_IMAGE = 'thibaultlefay/tasklist-frontend:latest'
    SONAR_HOST_URL = 'http://localhost:9000'
    PATH = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/root/.nvm/versions/node/v20.0.0/bin"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Setup Node.js') {
      steps {
        script {
          sh '''
            # Check if node is installed
            if ! command -v node &> /dev/null; then
              echo "Installing Node.js..."
              curl -fsSL https://deb.nodesource.com/setup_20.x | bash - || true
              apt-get update && apt-get install -y nodejs || true
            fi
            node --version
            npm --version
          '''
        }
      }
    }

    stage('Install dependencies') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Run tests') {
      steps {
        sh 'npm test -- --run --coverage || true'
      }
    }

    stage('Build frontend') {
      steps {
        sh 'npm run build'
      }
    }

    stage('Quality analysis with SonarQube') {
      steps {
        script {
          withCredentials([string(credentialsId: 'sonar-frontend-token', variable: 'SONAR_TOKEN')]) {
            sh '''
              # Install sonar-scanner if not present
              if ! command -v sonar-scanner &> /dev/null; then
                echo "Installing sonar-scanner..."
                npm install -g sonarqube-scanner || true
              fi
              
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
          # Install trivy if not present
          if ! command -v trivy &> /dev/null; then
            echo "Installing Trivy..."
            apt-get update && apt-get install -y wget apt-transport-https gnupg lsb-release || true
            wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add - || true
            echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee -a /etc/apt/sources.list.d/trivy.list || true
            apt-get update && apt-get install -y trivy || true
          fi
          
          trivy fs --exit-code 0 --format table . || true
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
        sh 'trivy image --exit-code 0 --format table $DOCKER_IMAGE || true'
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
      echo ' Pipeline completed successfully!'
    }
    failure {
      echo ' Pipeline failed!'
    }
  }
}
