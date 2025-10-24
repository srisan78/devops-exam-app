
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "sridhar76/devopsexamapp:latest"
        EKS_CLUSTER = "devopsapp"
        K8S_NAMESPACE = "devopsexamapp"
        AWS_REGION = "us-west-2"  // Update to your region
    }

    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/srisan78/devops-exam-app.git',
                    branch: 'master'
            }
        }

        stage('Verify Docker Compose') {
            steps {
                sh '''
                docker compose version || { echo "Docker Compose not available"; exit 1; }
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('backend') {
                    script {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh "docker build -t ${DOCKER_IMAGE} ."
                        }
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                #Clean up any existing containers
                docker compose down --remove-orphans || true

                #Start services
                docker compose up -d

                echo "Waiting for MySQL to be ready..."
                timeout 120s bash -c '
                while ! docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent;
                do
                    sleep 5;
                    docker compose logs mysql --tail=5 || true;
                done'

                sleep 10
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "=== Container Status ==="
                docker compose ps -a
                echo "=== Testing Flask Endpoint ==="
                curl -I http://localhost:5000 || true
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                        kubectl create secret docker-registry dockerhub-creds \
                            --docker-server=https://index.docker.io/v1/ \
                            --docker-username=kastrov \
                            --docker-password=\$(cat /var/jenkins_home/docker-creds/password) \
                            --namespace=${K8S_NAMESPACE} \
                            --dry-run=client -o yaml | kubectl apply -f -

                        kubectl apply -f deployment.yml
                        kubectl apply -f service.yml

                        kubectl rollout status deployment/devopsexamapp -n ${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo '🚀 Deployment successful!'
            sh 'docker compose ps'
        }
        failure {
            echo '❗ Pipeline failed. Check logs above.'
            sh 'docker compose logs --tail=50 || true'
        }
        always {
            sh 'docker compose logs --tail=20 || true'
        }
    }
}
