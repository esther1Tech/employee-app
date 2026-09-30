pipeline {
    agent any

    environment {
        AWS_REGION   = 'us-east-2'
        ECR_REGISTRY = '038304770452.dkr.ecr.us-east-2.amazonaws.com'
        ECR_BACKEND  = "${ECR_REGISTRY}/employee-backend"
        ECR_FRONTEND = "${ECR_REGISTRY}/employee-frontend"
        IMAGE_TAG    = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Test') {
            steps {
                sh '''
                    python3 -m pip install -r backend/requirements.txt
                    cd backend
                    DATABASE_URL=sqlite:///test.db python3 -m pytest -v
                '''
            }
        }

        stage('Build & Push') {
            steps {
                sh '''
                    echo "=== Logging in to AWS ECR ==="
                    aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin $ECR_REGISTRY

                    echo "=== Building and pushing backend ==="
                    docker build -t $ECR_BACKEND:$IMAGE_TAG backend/
                    docker push $ECR_BACKEND:$IMAGE_TAG
                    docker tag $ECR_BACKEND:$IMAGE_TAG $ECR_BACKEND:latest
                    docker push $ECR_BACKEND:latest

                    echo "=== Building and pushing frontend ==="
                    docker build -t $ECR_FRONTEND:$IMAGE_TAG frontend/
                    docker push $ECR_FRONTEND:$IMAGE_TAG
                    docker tag $ECR_FRONTEND:$IMAGE_TAG $ECR_FRONTEND:latest
                    docker push $ECR_FRONTEND:latest
                '''
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    ),
                    string(credentialsId: 'ec2-host', variable: 'EC2_HOST'),
                    string(credentialsId: 'database-url', variable: 'DATABASE_URL')
                ]) {
                    sh '''
                        aws ecr get-login-password --region "$AWS_REGION" | \
                        ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" "$SSH_USER@$EC2_HOST" \
                            "docker login --username AWS --password-stdin $ECR_REGISTRY"

                        printf '%s\n' "$DATABASE_URL" | \
                        ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" "$SSH_USER@$EC2_HOST" "
                            set -e
                            IFS= read -r DATABASE_URL
                            export DATABASE_URL
                            docker pull $ECR_BACKEND:$IMAGE_TAG
                            docker pull $ECR_FRONTEND:$IMAGE_TAG

                            docker network create employee-network 2>/dev/null || true
                            docker volume create employee-postgres-data >/dev/null

                            if ! docker inspect db >/dev/null 2>&1; then
                              docker run -d --name db --restart unless-stopped \
                                --network employee-network \
                                --mount source=employee-postgres-data,target=/var/lib/postgresql/data \
                                -e POSTGRES_DB=employees \
                                -e POSTGRES_USER=postgres \
                                -e POSTGRES_PASSWORD=postgres \
                                postgres:15
                            else
                              docker start db 2>/dev/null || true
                              docker network connect employee-network db 2>/dev/null || true
                            fi

                            for attempt in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30; do
                              if docker exec db pg_isready -U postgres -d employees >/dev/null 2>&1; then
                                break
                              fi
                              sleep 2
                            done
                            docker exec db pg_isready -U postgres -d employees

                            docker rm -f backend frontend 2>/dev/null || true

                            docker run -d --name backend --restart unless-stopped \
                              --network employee-network \
                              -p 5000:5000 \
                              -e DATABASE_URL \
                              $ECR_BACKEND:$IMAGE_TAG

                            docker run -d --name frontend --restart unless-stopped \
                              --network employee-network \
                              -p 80:80 \
                              $ECR_FRONTEND:$IMAGE_TAG
                        "
                    '''
                }
            }
        }
    }

    post {
        success { echo "Deployed build-${BUILD_NUMBER} to EC2 successfully" }
        failure { echo "Pipeline failed — check logs above" }
    }
}
