pipeline {
    agent any

    tools {
        jdk 'JDK'       // Ensure you configure JDK in Jenkins with this name
        nodejs 'NodeJS'    
    }

    parameters {
        string(name: 'ECR_REPO_NAME', defaultValue: 'my-ecr-repo', description: 'Enter your ECR repo name:')
        string(name: 'AWS_ACCOUNT_ID', defaultValue: '284044197248', description: 'Enter your AWS Account ID:')
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // Ensure this matches the configured name of the SonarQube scanner
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis: 2') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                     echo SONAR_HOST_URL=$SONAR_HOST_URL
                     ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=simple-web-app \
                        -Dsonar.projectName=Arts-web-app \
                        -Dsonar.sources=. \
                    """
                }
            }
        }

        stage('Quality Gates: 3') {
             steps {
                script {
                    timeout(time: 20, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted because Quality Gate status is: ${qg.status}"
                        }
                    }
                }
            }
        }


        stage('Docker Build: 6') {
            steps {
                sh """
                docker build -t ${params.ECR_REPO_NAME}-api:$BUILD_NUMBER -f ./api/Dockerfile ./api
                docker build -t ${params.ECR_REPO_NAME}-web:$BUILD_NUMBER -f ./web/Dockerfile ./web
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                trivy image ${params.ECR_REPO_NAME}-api:$BUILD_NUMBER
                trivy image ${params.ECR_REPO_NAME}-web:$BUILD_NUMBER
                """
            }
        }

        stage('Login to ECR & Tag and Push Image: 8') {
            steps {
                 withAWS(region: 'eu-central-1', credentials: 'sonar-cred') {
                    sh """
                    aws ecr get-login-password --region eu-central-1 \
                    | docker login --username AWS --password-stdin ${params.AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com
                    docker tag ${params.ECR_REPO_NAME}-api:$BUILD_NUMBER ${params.AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/${params.ECR_REPO_NAME}:$BUILD_NUMBER
                    docker tag ${params.ECR_REPO_NAME}-web:$BUILD_NUMBER ${params.AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/${params.ECR_REPO_NAME}:$BUILD_NUMBER

                    docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/${params.ECR_REPO_NAME}:$BUILD_NUMBER
                    docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.eu-central-1.amazonaws.com/${params.ECR_REPO_NAME}:$BUILD_NUMBER
                    """
                }
            }
        }

    }
}