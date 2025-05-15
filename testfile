pipeline {
    agent any

    parameters {
        choice(name: 'ENV', choices: ['dev', 'prod'], description: 'Deployment environment')
    }

    environment {
		GIT_REPO = 'https://github.com/snehakp0403/docker_nginx.git'
        ECR_REPO_NAME = 'ecr'
        IMAGE_NAME = "my-nginx-app:${params.ENV}"
		imageTag = "${ECR_REPO}:${params.ENV}"
        ECR_REPO = "public.ecr.aws/f9j3e3x1/ecr"
        CONTAINER_PORT = "80"
		
		AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
		AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
		AWS_DEFAULT_REGION = 'us-east-1'
    }

    stages {
		
		stage('Install AWS CLI') {
            steps {
                script {
                    sh '''
                        set -e
                        echo "Installing AWS CLI..."
                        sudo yum install -y unzip curl

                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        rm -rf aws
                        unzip -q awscliv2.zip
                        sudo ./aws/install --update
                        aws --version
                    '''
                }
            }
        }
		
        stage('Clone Repo') {
            steps {
                git url: "${GIT_REPO}", branch: 'main'
            }
        }

        stage('Test Dockerfile') {
            steps {
                sh """
                sudo docker build -t ${IMAGE_NAME} .
                sudo docker stop nginx-container || true
                sudo docker rm -f nginx-container || true
                sudo docker container prune -f
                sudo docker run -d --name nginx-container -p 8080:80 ${IMAGE_NAME}
                sleep 5
                STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080)
                if [ "\$STATUS" -ne 200 ]; then
                    echo "Health check failed"
                    exit 1
                fi
                """
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh """
						sudo docker tag $IMAGE_NAME ${ECR_REPO}:${params.ENV}
                        aws ecr-public get-login-password --region $AWS_DEFAULT_REGION | \
						docker login --username AWS --password-stdin public.ecr.aws
                        sudo docker push ${ECR_REPO}:${params.ENV}
                    """
                }
            }
        }

        stage('Deploy on Slave Node') {
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
            echo "Pipeline executed successfully"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
