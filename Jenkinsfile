
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        COMPOSE_PROJECT_NAME = 'skillbridge'
        FRONTEND_URL = 'http://host.docker.internal:3018'
        BACKEND_URL = 'http://host.docker.internal:8018'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "===== SkillBridge checkout ====="
                    git rev-parse --short HEAD
                    test -f backend/requirements.txt
                    test -f backend/Dockerfile
                    test -f frontend/Dockerfile
                    test -f docker-compose.yml
                    test -f Jenkinsfile
                '''
            }
        }

        stage('Backend Validation') {
            steps {
                sh '''
                    echo "===== Build backend validation image ====="
                    docker build -t skillbridge-backend-ci ./backend

                    echo "===== Validate Python syntax ====="
                    docker run --rm \
                      --entrypoint python \
                      skillbridge-backend-ci \
                      -m compileall -q /app

                    echo "Backend Python syntax validation passed."
                '''
            }
        }

        stage('Frontend Build Test') {
            steps {
                sh '''
                    echo "===== Build frontend ====="
                    docker build \
                      --target build \
                      -t skillbridge-frontend-ci \
                      ./frontend

                    echo "Frontend build passed."
                '''
            }
        }

        stage('Docker Compose Validation') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'skillbridge-mysql-root',
                        usernameVariable: 'MYSQL_ROOT_USERNAME',
                        passwordVariable: 'MYSQL_ROOT_PASSWORD'
                    ),
                    usernamePassword(
                        credentialsId: 'skillbridge-mysql-app',
                        usernameVariable: 'MYSQL_USER',
                        passwordVariable: 'MYSQL_PASSWORD'
                    ),
                    string(
                        credentialsId: 'skillbridge-jwt-secret',
                        variable: 'SECRET_KEY'
                    )
                ]) {
                    sh '''
                        set +x
                        docker compose config -q
                        echo "Docker Compose configuration is valid."
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'skillbridge-mysql-root',
                        usernameVariable: 'MYSQL_ROOT_USERNAME',
                        passwordVariable: 'MYSQL_ROOT_PASSWORD'
                    ),
                    usernamePassword(
                        credentialsId: 'skillbridge-mysql-app',
                        usernameVariable: 'MYSQL_USER',
                        passwordVariable: 'MYSQL_PASSWORD'
                    ),
                    string(
                        credentialsId: 'skillbridge-jwt-secret',
                        variable: 'SECRET_KEY'
                    )
                ]) {
                    sh '''
                        set +x
                        docker compose build
                    '''
                }
            }
        }

        stage('Deploy SkillBridge') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'skillbridge-mysql-root',
                        usernameVariable: 'MYSQL_ROOT_USERNAME',
                        passwordVariable: 'MYSQL_ROOT_PASSWORD'
                    ),
                    usernamePassword(
                        credentialsId: 'skillbridge-mysql-app',
                        usernameVariable: 'MYSQL_USER',
                        passwordVariable: 'MYSQL_PASSWORD'
                    ),
                    string(
                        credentialsId: 'skillbridge-jwt-secret',
                        variable: 'SECRET_KEY'
                    )
                ]) {
                    sh '''
                        set +x
                        docker compose up -d --remove-orphans
                        docker compose ps
                    '''
                }
            }
        }

        stage('Deployment Verification') {
            steps {
                sh '''
                    set -eu

                    echo "===== Verify SkillBridge endpoints ====="

                    docker run --rm \
                      --entrypoint sh \
                      curlimages/curl:8.12.1 \
                      -c '
                        set -eu

                        check_url() {
                            name="$1"
                            url="$2"
                            attempt=1

                            while [ "$attempt" -le 30 ]; do
                                if curl -fsS --max-time 5 "$url" -o /dev/null; then
                                    echo "$name is responding: $url"
                                    return 0
                                fi

                                echo "Waiting for $name ($attempt/30)..."
                                attempt=$((attempt + 1))
                                sleep 5
                            done

                            echo "ERROR: $name did not respond: $url"
                            return 1
                        }

                        check_url Frontend http://host.docker.internal:3018/
                        check_url Backend http://host.docker.internal:8018/
                        check_url API-docs http://host.docker.internal:8018/docs
                      '
                '''
            }
        }
    }

    post {
        success {
            echo 'SkillBridge pipeline completed successfully.'

            echo 'Frontend: http://localhost:3018'
            echo 'Backend:  http://localhost:8018'
            echo 'API docs: http://localhost:8018/docs'
        }

        failure {
            echo 'SkillBridge pipeline failed. Inspect the failed stage.'

            sh '''
                docker ps -a --filter "name=skillbridge" || true
                docker logs --tail 80 skillbridge-backend-1 || true
                docker logs --tail 80 skillbridge-frontend-1 || true
                docker logs --tail 80 skillbridge-mysql-1 || true
            '''
        }

        always {
            echo 'SkillBridge pipeline finished.'
        }
    }
}
