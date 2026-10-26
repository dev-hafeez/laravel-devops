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

                        sed -i '/^APP_KEY=/d' .env
                        printf 'APP_KEY=%s\\n' "$APP_KEY_VALUE" >> .env

                        sed -i '/^DB_PASSWORD=/d' .env
                        printf 'DB_PASSWORD=%s\\n' "$DB_PASSWORD_VALUE" >> .env

                        sed -i '/^DB_ROOT_PASSWORD=/d' .env
                        printf 'DB_ROOT_PASSWORD=%s\\n' "$DB_ROOT_PASSWORD_VALUE" >> .env

                        sed -i 's/^DB_CONNECTION=.*/DB_CONNECTION=mysql/' .env
                        sed -i 's/^DB_HOST=.*/DB_HOST=db/' .env
                        sed -i 's/^DB_PORT=.*/DB_PORT=3306/' .env

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
                    for i in $(seq 1 30); do
                        if docker compose exec -T db \
                            mysqladmin ping -h 127.0.0.1 \
                            --silent; then
                            echo "Database is ready."
                            exit 0
                        fi

                        echo "Waiting for database..."
                        sleep 2
                    done

                    echo "Database did not become ready."
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

                    echo "Laravel deployment verification completed."
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
