pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = 'laravel-devops'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Environment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'app-key',
                        variable: 'APP_KEY_VALUE'
                    ),
                    string(
                        credentialsId: 'db-password',
                        variable: 'DB_PASSWORD_VALUE'
                    ),
                    string(
                        credentialsId: 'db-root-password',
                        variable: 'DB_ROOT_PASSWORD_VALUE'
                    )
                ]) {
                    sh '''
                        set +x

                        cp .env.example .env

                        set_env() {
                            key="$1"
                            val="$2"
                            if grep -q "^#\\? *${key}=" .env; then
                                sed -i "s|^#\\? *${key}=.*|${key}=${val}|" .env
                            else
                                printf '%s=%s\\n' "$key" "$val" >> .env
                            fi
                        }

                        set_env "APP_KEY" "$APP_KEY_VALUE"
                        set_env "DB_CONNECTION" "mysql"
                        set_env "DB_HOST" "db"
                        set_env "DB_PORT" "3306"
                        set_env "DB_DATABASE" "laravel"
                        set_env "DB_USERNAME" "laravel_user"
                        set_env "DB_PASSWORD" "$DB_PASSWORD_VALUE"
                        set_env "DB_ROOT_PASSWORD" "$DB_ROOT_PASSWORD_VALUE"

                        chmod 644 .env
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker compose build app
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    docker compose up -d --build
                    docker compose ps
                '''
            }
        }

        stage('Wait for Database') {
            steps {
                sh '''
                    echo "Waiting for database and network readiness from app container..."
                    for i in $(seq 1 30); do
                        if docker compose exec -T app php -r "
                            try {
                                \\$pdo = new PDO('mysql:host=db;port=3306;dbname=laravel', 'laravel_user', getenv('DB_PASSWORD'), [
                                    PDO::ATTR_TIMEOUT => 3,
                                    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                                ]);
                                exit(0);
                            } catch (Throwable \\$e) {
                                exit(1);
                            }
                        "; then
                            echo "Database connection and DNS resolution verified from app container."
                            exit 0
                        fi

                        echo "Waiting for database connectivity from app (attempt $i/30)..."
                        sleep 2
                    done

                    echo "Database did not become ready for app container."
                    exit 1
                '''
            }
        }

        stage('Database Migration') {
            steps {
                sh '''
                    docker compose exec -T app php artisan config:clear
                    docker compose exec -T app php artisan migrate --force
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    docker compose exec -T app php artisan --version
                    docker compose ps

                    echo "Verifying application HTTP accessibility..."
                    for i in $(seq 1 15); do
                        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000 || true)
                        if [ "$STATUS" = "200" ]; then
                            echo "Application is accessible and returned HTTP $STATUS."
                            exit 0
                        fi
                        echo "Waiting for HTTP 200 response (got: $STATUS, attempt $i/15)..."
                        sleep 2
                    done

                    echo "Application failed accessibility check (status: $STATUS)."
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }

        failure {
            echo 'Deployment failed. Check the Jenkins console output.'
        }

        always {
            sh 'docker compose ps || true'
        }
    }
}
