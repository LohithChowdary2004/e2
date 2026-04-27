pipeline {
    agent any

    triggers {
        // Requires GitHub webhook + Jenkins GitHub integration.
        githubPush()
    }

    environment {
        COMPOSE_PROJECT_NAME = "docker_master"
        COMPOSE_FILE = "docker-compose.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Availability Check') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'docker --version'
                        sh 'docker compose version'
                    } else {
                        bat 'docker --version'
                        bat 'docker compose version'
                    }
                }
            }
        }

        stage('Clean Previous Containers') {
            steps {
                echo 'Stopping old containers (if any)...'
                script {
                    if (isUnix()) {
                        sh 'docker compose down --remove-orphans || true'
                    } else {
                        // Avoid hard failure if nothing is running.
                        bat 'docker compose down --remove-orphans || exit /b 0'
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images...'
                script {
                    if (isUnix()) {
                        sh 'docker compose build --no-cache'
                    } else {
                        bat 'docker compose build --no-cache'
                    }
                }
            }
        }

        stage('Run Docker Containers') {
            steps {
                echo 'Starting containers...'
                script {
                    if (isUnix()) {
                        sh 'docker compose up -d'
                    } else {
                        bat 'docker compose up -d'
                    }
                }
            }
        }

        stage('Check Running Services') {
            steps {
                echo 'Checking running containers...'
                script {
                    if (isUnix()) {
                        sh 'docker compose ps'
                    } else {
                        bat 'docker compose ps'
                    }
                }
            }
        }

        stage('Backend Health Check') {
            steps {
                echo 'Validating backend endpoint...'
                script {
                    if (isUnix()) {
                        sh '''
                            for i in 1 2 3 4 5; do
                              code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/captcha/generate || true)
                              if [ "$code" = "200" ]; then
                                echo "Backend health check passed."
                                exit 0
                              fi
                              echo "Attempt $i failed with status: $code"
                              sleep 5
                            done
                            echo "Backend health check failed."
                            exit 1
                        '''
                    } else {
                        powershell '''
                            $ok = $false
                            for ($i = 1; $i -le 5; $i++) {
                                try {
                                    $response = Invoke-WebRequest -Uri "http://localhost:8080/captcha/generate" -UseBasicParsing -TimeoutSec 10
                                    if ($response.StatusCode -eq 200) {
                                        Write-Host "Backend health check passed."
                                        $ok = $true
                                        break
                                    }
                                } catch {
                                    Write-Host "Attempt $i failed: $($_.Exception.Message)"
                                }
                                Start-Sleep -Seconds 5
                            }
                            if (-not $ok) {
                                throw "Backend health check failed."
                            }
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up containers after build...'
            script {
                if (isUnix()) {
                    sh 'docker compose down --remove-orphans || true'
                } else {
                    bat 'docker compose down --remove-orphans || exit /b 0'
                }
            }
        }
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed. Please check logs.'
            script {
                if (isUnix()) {
                    sh 'docker compose logs --tail=200'
                } else {
                    bat 'docker compose logs --tail=200'
                }
            }
        }
    }
}