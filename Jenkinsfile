pipeline {
  agent any

  environment {
    AWS_REGION = 'ap-south-1'
    ECR_REGISTRY = '486991249088.dkr.ecr.ap-south-1.amazonaws.com'
    BACKEND_REPO = "${ECR_REGISTRY}/blog-backend"
    FRONTEND_REPO = "${ECR_REGISTRY}/blog-frontend"
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/deep1172/Blog-app.git'
      }
    }

    stage('Test') {
      steps {
        dir('backend') {
          sh 'npm install && npm test'
        }
        dir('frontend') {
          sh 'npm install && npm test'
        }
      }
    }

    stage('Login to ECR') {
      steps {
        sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
      }
    }

    stage('Build Docker Images') {
      steps {
        sh 'docker build -t blog-backend:latest ./backend'  
        sh 'docker build -t blog-frontend:latest ./frontend'
      }
    }

    stage('Push Docker Images to ECR') {
      steps {
        sh '''
          docker tag blog-backend:latest $BACKEND_REPO:latest
          docker tag blog-frontend:latest $FRONTEND_REPO:latest
          docker push $BACKEND_REPO:latest
          docker push $FRONTEND_REPO:latest
        '''
      }
    }

    stage('Terraform Deploy') {
      steps {
        dir('infra') {
          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {        
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
        echo "Running health check after deploy..."
        // Optional: curl ALB URL to verify deployment      
      }
    }
  }

  post {
    failure {
      echo "❌ Pipeline failed. You can trigger rollback or notify here."
    }
    success {
      echo "✅ Pipeline executed successfully."
    }
  }
}
