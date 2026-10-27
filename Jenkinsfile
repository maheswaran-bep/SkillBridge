
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
                '''
            }
        }

        stage('Backend Validation') {
            steps {
                sh '''
                    python3 -m compileall -q backend
                    echo "Backend Python syntax validation passed."
                '''
            }
        }

        stage('Frontend Build Test') {
            steps {
                dir('frontend') {
                    sh '''
                        npm ci
                        npm run build
                    '''
                }
            }
        }

        stage('Docker Compose Validation') {
            steps {
                sh 'docker compose config -q'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
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
                        docker compose config -q
                        docker compose up -d --remove-orphans
                        docker compose ps
                    '''
                }
            }
        }

        stage('Deployment Verification') {
            steps {
                sh '''
                    python3 - <<'PY'
import time
import urllib.request

targets = {
    "Frontend": "http://host.docker.internal:3018/",
    "Backend": "http://host.docker.internal:8018/",
    "API docs": "http://host.docker.internal:8018/docs",
}

for name, url in targets.items():
    for attempt in range(30):
        try:
            with urllib.request.urlopen(url, timeout=5) as response:
                if response.status < 200 or response.status >= 400:
                    raise RuntimeError(
                        f"{name} returned HTTP {response.status}"
                    )
                print(f"{name}: HTTP {response.status} - {url}")
                break
        except Exception:
            if attempt == 29:
                raise
            time.sleep(5)
PY
                '''
            }
        }
    }

    post {
        success {
            echo 'SkillBridge deployment completed successfully.'
        }
        failure {
            echo 'SkillBridge pipeline failed. Check the stage logs.'
            sh 'docker compose ps || true'
            sh 'docker compose logs --tail=80 backend mysql frontend || true'
        }
        always {
            echo 'SkillBridge pipeline finished.'
        }
    }
}
