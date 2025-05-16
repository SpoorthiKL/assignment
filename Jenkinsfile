pipeline {
    agent { label 'label' }

    parameters {
        choice(name: 'ENV', choices: ['dev', 'prod'], description: 'Deployment environment')
    }

    environment {
        GIT_REPO = 'https://github.com/snehakp0403/docker_nginx.git'
        ECR_REPO = "public.ecr.aws/h0n6v7u5/jenecr"
        CONTAINER_PORT = "80"
    }

    stages {
        stage('AWS CLI Installation') {
            steps {
                sh '''
                    set -e
                    echo "Installing AWS CLI..."
                    sudo apt update
                    sudo apt install -y unzip curl
                    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                    rm -rf aws
                    unzip -q awscliv2.zip
                    sudo ./aws/install --update
                    aws --version
                '''
            }
        }

        stage('Cloning the Repository') {
            steps {
                git url: "${GIT_REPO}", branch: 'main'
            }
        }

        stage('Dockerfile Testing') {
            steps {
                script {
                    def imageName = "my-nginx-app:${params.ENV}"
                    sh """
                        sudo docker build -t ${imageName} .
                        sudo docker stop nginx-container || true
                        sudo docker rm -f nginx-container || true
                        sudo docker container prune -f
                        sudo docker run -d --name nginx-container -p 8080:80 ${imageName}
                        sleep 5
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080)
                        if [ "\$STATUS" -ne 200 ]; then
                            echo "Health check failed"
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Pushing the docker image to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    script {
                        def imageName = "my-nginx-app:${params.ENV}"
                        def imageTag = "${ECR_REPO}:${params.ENV}"

                        sh """
                            # Clean any cached Docker credentials
                            rm -f ~/.docker/config.json

                            # Export AWS creds for aws cli and docker login
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=us-east-1

                            # Verify identity
                            aws sts get-caller-identity

                            # Tag and login
                            sudo docker tag ${imageName} ${imageTag}
                            aws ecr-public get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin public.ecr.aws

                            # Push image
                            sudo docker push ${imageTag}
                        """
                    }
                }
            }
        }

        stage('Deploying on jenkins agent Slave') {
            steps {
                script {
                    def imageTag = "${ECR_REPO}:${params.ENV}"
                    sh """
                        sudo docker stop nginx-container || true
                        sudo docker rm -f nginx-container || true
                        sudo docker container prune -f
                        sudo docker run -d --name nginx-container -p 8080:80 ${imageTag}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline is sucessful with image being pushed to ECR"
        }
        failure {
            echo "Pipeline is failing"
        }
    }
}
