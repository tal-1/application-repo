pipeline {
    agent none
    
    environment {
        ECR_URL = "992382545251.dkr.ecr.us-east-1.amazonaws.com/molcho-ecr/calculator-app"
        REGION  = "us-east-1"
    }

    stages {
        stage('Build & Test') {
            agent { docker { image 'python:3.9-slim' } }
            steps {
                script {
                    sh "pip install -r requirements.txt pytest"
                    sh "pytest tests/" 
                }
            }
        }

        stage('Build & Push Image') {
            agent { label 'built-in' }
            when { changeRequest() } 
            steps {
                script {
                    def tag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                    
                    sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_URL}"

                    sh "docker build -t ${ECR_URL}:${tag} ."

                    sh "docker push ${ECR_URL}:${tag}"
                    
                    echo "Pushed image: ${ECR_URL}:${tag}"
                }
            }
        }
    }
}
