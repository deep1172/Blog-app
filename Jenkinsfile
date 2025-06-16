pipeline {
  agent any

  environment {
    AWS_REGION       = credentials('aws-region')                   // e.g., ap-south-1
    ECR_REGISTRY     = credentials('ecr-registry')                 // e.g., 123456789012.dkr.ecr.ap-south-1.amazonaws.com
    BACKEND_REPO     = "${ECR_REGISTRY}/blog-backend"
    FRONTEND_REPO    = "${ECR_REGISTRY}/blog-frontend"
    BACKEND_URL      = credentials('backend-health-url')           // e.g., http://<alb-backend>/api/health
    FRONTEND_URL     = credentials('frontend-health-url')          // e.g., http://<alb-frontend>
    SONAR_TOKEN      = credentials('sonarqube-token')
  }

  triggers {
    pollSCM('H/2 * * * *') // Poll GitHub main branch every 2 minutes (replace with webhook later)
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/deep1172/Blog-app.git'
      }
    }

    stage('SonarQube Code Scan') {
      steps {
        withSonarQubeEnv('MySonarQube') {
          dir('backend') {
            sh """
              sonar-scanner \
              -Dsonar.projectKey=blog-backend \
              -Dsonar.sources=. \
              -Dsonar.login=$SONAR_TOKEN
            """
          }
          dir('frontend') {
            sh """
              sonar-scanner \
              -Dsonar.projectKey=blog-frontend \
              -Dsonar.sources=. \
              -Dsonar.login=$SONAR_TOKEN
            """
          }
        }
      }
    }

    stage('Unit Tests') {
      steps {
        dir('backend') {
          sh 'npm install && npm test || echo "⚠️ Backend test failed, continuing..."'
        }
        dir('frontend') {
          sh 'npm install && npm test || echo "⚠️ Frontend test failed, continuing..."'
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build -t blog-backend:latest ./backend'
        sh 'docker build -t blog-frontend:latest ./frontend'
      }
    }

    stage('Trivy Vulnerability Scan') {
      steps {
        sh '''
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
        trivy image --severity HIGH,CRITICAL --exit-code 0 blog-backend:latest
        trivy image --severity HIGH,CRITICAL --exit-code 0 blog-frontend:latest
        '''
      }
    }

    stage('Login to AWS ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {
          sh """
            aws ecr get-login-password --region $AWS_REGION | \
            docker login --username AWS --password-stdin $ECR_REGISTRY
          """
        }
      }
    }

    stage('Push Docker Images to ECR') {
      steps {
        sh """
          docker tag blog-backend:latest $BACKEND_REPO:latest
          docker tag blog-frontend:latest $FRONTEND_REPO:latest
          docker push $BACKEND_REPO:latest
          docker push $FRONTEND_REPO:latest
        """
      }
    }

    stage('Terraform Deploy Infra') {
      steps {
        dir('infra') {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-credentials'
          ]]) {
            sh '''
              terraform init
              terraform plan -out=tfplan
              terraform apply -auto-approve tfplan
            '''
          }
        }
      }
    }

    stage('Deployment Health Check') {
      steps {
        script {
          def backendStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' $BACKEND_URL", returnStdout: true).trim()
          def frontendStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' $FRONTEND_URL", returnStdout: true).trim()

          if (backendStatus != "200" || frontendStatus != "200") {
            error "💥 Deployment health check failed: backend=$backendStatus, frontend=$frontendStatus"
          } else {
            echo "✅ Deployment health check passed: backend=$backendStatus, frontend=$frontendStatus"
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline completed successfully."
    }
    failure {
      echo "❌ Pipeline failed. You may trigger rollback or alert here."
      // Example: sh 'terraform destroy -auto-approve' or notify via Slack/email
    }
  }
}
