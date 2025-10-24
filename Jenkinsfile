pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "sridhar76/devopsexamapp:latest"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/srisan78/devops-exam-app.git', branch: 'master'
            }
        }

        stage('Verify Docker Compose') {
            steps {
                sh '''
                if ! command -v docker >/dev/null 2>&1; then
                  echo "docker not found"
                  exit 1
                fi

                # Accept either "docker compose" (new CLI) or "docker-compose" (legacy)
                if ! docker compose version >/dev/null 2>&1 && ! docker-compose version >/dev/null 2>&1; then
                  echo "Docker Compose not available"
                  exit 1
                fi

                docker --version
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('backend') {
                    script {
                        // Ensure we are logged in to the registry while building (if required)
                        withDockerRegistry([credentialsId: 'docker', url: 'https://index.docker.io/v1/']) {
                            sh "docker build -t ${DOCKER_IMAGE} ."
                        }
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker', url: 'https://index.docker.io/v1/']) {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                script {
                    // Stop any previous run, then start services and wait for MySQL
                    sh '''
                    set -o errexit
                    docker compose down --remove-orphans || true
                    docker compose up -d
                    '''
                    timeout(time: 120, unit: 'SECONDS') {
                        sh '''
                        echo "Waiting for MySQL to be ready..."
                        until docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent >/dev/null 2>&1; do
                          sleep 5
                          docker compose logs mysql --tail=5 || true
                        done
                        '''
                    }
                    sh 'sleep 5'
                }
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
    }
}
