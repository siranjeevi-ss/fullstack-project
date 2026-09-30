pipeline {
    agent any

    environment {
        FRONTEND_DIR = "frontend"
        BACKEND_DIR  = "backend"

        BACKEND_IMAGE  = "fullstack-project-backend"
        FRONTEND_IMAGE = "fullstack-project-frontend"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend') {
            steps {
                dir("${BACKEND_DIR}") {
                    sh '''
                    rm -rf venv

                    python3.11 -m venv venv

                    ./venv/bin/pip install --upgrade pip

                    ./venv/bin/pip install -r requirements.txt
                    '''
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir("${FRONTEND_DIR}") {
                    sh '''
                    npm install
                    npm run build
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                docker build --no-cache -t ${BACKEND_IMAGE}:latest ${BACKEND_DIR}
                docker build -t ${FRONTEND_IMAGE}:latest ${FRONTEND_DIR}
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                echo "Scanning Backend Image..."
                trivy image \
                    --severity HIGH,CRITICAL \
                    --ignore-unfixed \
                    --exit-code 1 \
                    ${BACKEND_IMAGE}:latest

                echo "Scanning Frontend Image..."
                trivy image \
                    --severity HIGH,CRITICAL \
                    --ignore-unfixed \
                    --exit-code 1 \
                    ${FRONTEND_IMAGE}:latest
                '''
            }
        }

        stage('Start Backend') {
            steps {
                sh '''
                pkill -f "uvicorn" || true

                cd backend

                nohup ./venv/bin/uvicorn main:app \
                    --host 0.0.0.0 \
                    --port 8000 \
                    > backend.log 2>&1 &
                '''
            }
        }

        stage('Start Frontend') {
            steps {
                sh '''
                pkill -f "vite preview" || true

                cd frontend

                nohup npm run preview -- \
                    --host 0.0.0.0 \
                    --port 5173 \
                    > frontend.log 2>&1 &
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                sleep 10

                echo "Checking Backend..."
                curl -f http://127.0.0.1:8000/

                echo "Checking Frontend..."
                curl -f http://127.0.0.1:5173/

                echo "Health Check Passed"
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment Successful"
        }

        failure {
            echo "Deployment Failed"
        }

        always {
            sh '''
            echo "Cleaning temporary Python virtual environment..."
            rm -rf backend/venv
            '''
        }
    }
}
