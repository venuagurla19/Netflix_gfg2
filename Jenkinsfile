pipeline {
  agent any

  environment {
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials') // ID containing username/password
    DB_PASSWORD = credentials('db-password') // DB password from Jenkins credentials
    DB_HOST = 'database-1.ctsgs6ymgs9r.ap-south-1.rds.amazonaws.com'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          # Install Node.js properly using NodeSource
          curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
          yum install -y nodejs
          node -v
          npm install
          npm test
        '''
      }
    }

    stage('Docker Build and Push') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
            def backendImage = docker.build("${DOCKERHUB_CREDENTIALS_USR}/node-backend", 'node-backend')
            def frontendImage = docker.build("${DOCKERHUB_CREDENTIALS_USR}/node-frontend", 'html')

            backendImage.push("latest")
            frontendImage.push("latest")
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          kubectl apply -f deploy/backend-configmap.yaml
          kubectl apply -f deploy/backend-secret.yaml
          kubectl apply -f deploy/backend-deployment.yaml
          kubectl apply -f deploy/backend-service.yaml
        '''
      }
    }

    stage('Update Frontend Config') {
      steps {
        script {
          def apiUrl = sh(
            script: "kubectl get svc node-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
            returnStdout: true
          ).trim()

          echo "Backend API URL: ${apiUrl}"

          // Replace placeholder and apply config
          sh "sed 's|\\\${API_URL}|${apiUrl}|g' deploy/webapp-config.yaml > updated-webapp.yaml"
          sh 'kubectl apply -f updated-webapp.yaml'
        }
      }
    }

    stage('Deploy Frontend') {
      steps {
        sh '''
          kubectl apply -f deploy/webapp-deployment.yaml
          kubectl apply -f deploy/webapp-service.yaml
        '''
      }
    }

    stage('Verify Deployment') {
      steps {
        sh '''
          echo "Verifying pod status..."
          kubectl get pods
        '''
      }
    }
  }

  post {
    success {
      echo '✅ Pipeline completed successfully.'
    }
    failure {
      echo '❌ Pipeline failed. Please check the logs above.'
    }
  }
}
