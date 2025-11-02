pipeline {
  agent any

  environment {
    AWS_REGION = 'us-east-1'
    AWS_ACCOUNT_ID = '123456789012'
    API_REPOSITORY = 'garden-api'
    WEB_REPOSITORY = 'garden-web'
    SONAR_PROJECT_KEY = 'replace-with-sonarqube-project-key'
    SONAR_PROJECT_NAME = 'Garden Three Tier App'
    TRIVY_SEVERITY = 'CRITICAL,HIGH'
    TRIVY_IMAGE = 'aquasec/trivy:0.48.3'
  }

  options {
    skipDefaultCheckout(true)
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Build Metadata') {
      steps {
        script {
          env.GIT_COMMIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          env.ECR_REGISTRY = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
          env.API_IMAGE = "${env.ECR_REGISTRY}/${env.API_REPOSITORY}:${env.GIT_COMMIT_SHORT}"
          env.API_IMAGE_LATEST = "${env.ECR_REGISTRY}/${env.API_REPOSITORY}:latest"
          env.WEB_IMAGE = "${env.ECR_REGISTRY}/${env.WEB_REPOSITORY}:${env.GIT_COMMIT_SHORT}"
          env.WEB_IMAGE_LATEST = "${env.ECR_REGISTRY}/${env.WEB_REPOSITORY}:latest"
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh """
            sonar-scanner \
              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
              -Dsonar.projectName=${SONAR_PROJECT_NAME} \
              -Dsonar.projectVersion=${GIT_COMMIT_SHORT} \
              -Dsonar.sources=. \
              -Dsonar.host.url=$SONAR_HOST_URL
          """
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Docker Images') {
      steps {
        sh """
          docker build -t ${API_IMAGE} -f api/Dockerfile api
          docker build -t ${WEB_IMAGE} -f web/Dockerfile web
        """
        sh """
          docker tag ${API_IMAGE} ${API_IMAGE_LATEST}
          docker tag ${WEB_IMAGE} ${WEB_IMAGE_LATEST}
        """
      }
    }

    stage('Trivy Scan') {
      steps {
        script {
          [env.API_IMAGE, env.WEB_IMAGE].each { imageName ->
            sh """
              docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                ${TRIVY_IMAGE} image \
                --exit-code 1 \
                --severity ${TRIVY_SEVERITY} \
                ${imageName}
            """
          }
        }
      }
    }

    stage('Push Images to ECR') {
      steps {
        withCredentials([
          [$class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-ecr-credentials',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
        ]) {
          sh """
            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

            aws ecr describe-repositories --repository-names ${API_REPOSITORY} --region ${AWS_REGION} || \
              aws ecr create-repository --repository-name ${API_REPOSITORY} --region ${AWS_REGION}

            aws ecr describe-repositories --repository-names ${WEB_REPOSITORY} --region ${AWS_REGION} || \
              aws ecr create-repository --repository-name ${WEB_REPOSITORY} --region ${AWS_REGION}

            docker push ${API_IMAGE}
            docker push ${API_IMAGE_LATEST}
            docker push ${WEB_IMAGE}
            docker push ${WEB_IMAGE_LATEST}
          """
        }
      }
    }
  }

  post {
    always {
      sh 'docker image prune -f || true'
      cleanWs()
    }
  }
}
