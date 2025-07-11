pipeline {
  agent any
  environment {
    
    AWS_REGION      = credentials('aws-region')
    ECR_REGISTRY    = credentials('ecr-registry')
    BACKEND_REPO    = "${env.ECR_REGISTRY}/blog-backend"
    FRONTEND_REPO   = "${env.ECR_REGISTRY}/blog-frontend"

  }

  options {
    skipStagesAfterUnstable()
  }

  stages {
    stage('Verify Branch') {
      when {
        anyOf {
          expression { env.GIT_BRANCH == 'origin/main' }
          expression { env.BRANCH_NAME == 'main' }
        }
      }
      steps {
        echo "✅ Running pipeline on main branch"
      }
    }

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/deep1172/Blog-app.git'
      }
    }

    stage('SonarQube - Backend') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          withSonarQubeEnv('MySonarQube') {
            dir('backend') {
              sh '''
                sonar-scanner \
                  -Dsonar.projectKey=blog-backend \
                  -Dsonar.sources=. \
                  -Dsonar.login=$SONAR_TOKEN \
                  -Dsonar.working.directory=.scannerwork-backend
              '''
            }
          }
        }
      }
    }

    stage('SonarQube - Frontend') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          withSonarQubeEnv('MySonarQube') {
            dir('frontend') {
              sh '''
                sonar-scanner \
                  -Dsonar.projectKey=blog-frontend \
                  -Dsonar.sources=. \
                  -Dsonar.login=$SONAR_TOKEN \
                  -Dsonar.working.directory=.scannerwork-frontend
              '''
            }
          }
        }
      }
    }


    stage('Docker Build') {
      steps {
        sh 'docker build -t blog-backend:latest ./Backend'
        sh 'docker build -t blog-frontend:latest ./Frontend'
      }
    }

  stage('Trivy Vulnerability Scan') {
    steps {
      sh '''
        mkdir -p ~/.local/bin
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b ~/.local/bin
        export PATH="$HOME/.local/bin:$PATH"

        trivy version

        trivy image --severity HIGH,CRITICAL --exit-code 0 blog-backend:latest
        trivy image --severity HIGH,CRITICAL --exit-code 0 blog-frontend:latest

        trivy image -f json -o trivy-backend-report.json blog-backend:latest

        trivy image -f json -o trivy-frontend-report.json blog-frontend:latest
    '''
  }
}

stage('Login to AWS ECR') {
  steps {
    withCredentials([
      [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']
    ]) {
      sh '''
        echo "🔐 Logging into AWS ECR..."
        aws ecr get-login-password --region "$AWS_REGION" | \
        docker login --username AWS --password-stdin "$ECR_REGISTRY"
      '''
    }
  }
}

stage('Push Docker Images to ECR') {
  steps {
    sh '''
      echo "📦 Tagging and pushing Docker images..."
      docker tag blog-backend:latest "$BACKEND_REPO:latest"
      docker tag blog-frontend:latest "$FRONTEND_REPO:latest"
      docker push "$BACKEND_REPO:latest"
      docker push "$FRONTEND_REPO:latest"
    '''
  }
}

  }

  post {
    success {
      echo "✅ Pipeline completed successfully."
     
    }
    failure {
      echo " Pipeline failed."
      echo "Rollback or manual inspection may be required."
     
    }
  }
}