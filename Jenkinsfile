pipeline {
  agent any

  environment {
    
    AWS_REGION      = credentials('aws-region')
    ECR_REGISTRY    = credentials('ecr-registry')
    BACKEND_REPO    = "${env.ECR_REGISTRY}/blog-backend"
    FRONTEND_REPO   = "${env.ECR_REGISTRY}/blog-frontend"
    BACKEND_URL     = credentials('backend-health-url')
    FRONTEND_URL    = credentials('frontend-health-url')
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
                  -Dsonar.host.url=http://13.126.169.178:9000 \
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
                  -Dsonar.host.url=http://13.126.169.178:9000 \
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
    withCredentials([[ 
      $class: 'AmazonWebServicesCredentialsBinding', 
      credentialsId: 'aws-credentials' 
    ]]) {
      sh '''
        echo "🔐 Attempting ECR login..."
        echo "Region: $AWS_REGION"
        echo "Registry: $ECR_REGISTRY"

        login_cmd=$(aws ecr get-login-password --region "$AWS_REGION")
        if [ -z "$login_cmd" ]; then
          echo "❌ Failed to get login password from AWS ECR"
          exit 1
        fi

        echo "$login_cmd" | docker login --username AWS --password-stdin "$ECR_REGISTRY"

        if [ $? -ne 0 ]; then
          echo "❌ Docker login to ECR failed"
          exit 1
        fi

        echo "✅ Logged into ECR successfully"
      '''
    }
  }
}

    stage('Push Docker Images to ECR') {
      steps {
        sh '''
          docker tag blog-backend:latest "$BACKEND_REPO:latest"
          docker tag blog-frontend:latest "$FRONTEND_REPO:latest"
          docker push "$BACKEND_REPO:latest"
          docker push "$FRONTEND_REPO:latest"
        '''
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
            error "💥 Health check failed: backend=$backendStatus, frontend=$frontendStatus"
          } else {
            echo "✅ Health check passed: backend=$backendStatus, frontend=$frontendStatus"
            echo "🌐 Access your application here: http://app-alb-189786761.ap-south-1.elb.amazonaws.com/"
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline completed successfully."
      echo "🚀 Application is live at: http://app-alb-189786761.ap-south-1.elb.amazonaws.com/"
    }
    failure {
      echo "❌ Pipeline failed."
      echo "ℹ️ Rollback or manual inspection may be required."
      echo "🔗 You can still check: http://app-alb-189786761.ap-south-1.elb.amazonaws.com/ (if infra was partially up)"
    }
  }
}