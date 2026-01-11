pipeline {
    agent none
    environment {

        ECR_URL = "992382545251.dkr.ecr.us-east-1.amazonaws.com/molcho-ecr"
        REGION  = "us-east-1"
        PROD_IP = "54.158.101.162"
    }

    stages {
        stage('CI: Build & Test') {
            agent { docker { image 'python:3.9-slim' } }
            steps {
                script {
                    sh "pip install -r requirements.txt pytest"
                    sh 'export PYTHONPATH=$PYTHONPATH:. && pytest tests/'
                }
            }
        }


        stage('CI: Build & Push Image') {
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


        stage('CD: Promote & Push') {
            agent { label 'built-in' }
            when { branch 'main' }
            steps {
                script {
                    def commitHash = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    def prodTag = "prod-${commitHash}"

                    sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_URL}"

                    sh "docker build -t ${ECR_URL}:${prodTag} -t ${ECR_URL}:latest ."
                    sh "docker push ${ECR_URL}:${prodTag}"
                    sh "docker push ${ECR_URL}:latest"
                }
            }
        }

        stage('CD: Deploy to Production') {
            agent { label 'built-in' }
            when { branch 'main' }
            steps {
                sshagent(['prod-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@${PROD_IP} "
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_URL} &&
                            docker pull ${ECR_URL}:latest &&
                            docker stop calculator || true &&
                            docker rm calculator || true &&
                            docker run -d --name calculator -p 80:5000 ${ECR_URL}:latest
                        "
                    """
                }
            }
        }

        stage('CD: Health Verification') {
            agent { label 'built-in' }
            when { branch 'main' }
            steps {
                script {
                    retry(5) {
                        sh "sleep 10"
                    }   sh "curl --fail http://${PROD_IP}:80/health"
                }
            }
        }
    }
}
