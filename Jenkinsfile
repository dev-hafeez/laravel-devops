pipeline {
    agent any

    triggers {
        pollSCM('H/5 * * * *')
    }

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
                        credentialsId: 'db-password',
                        variable: 'DB_PASSWORD'
                    ),
                    string(
                        credentialsId: 'db-root-password',
                        variable: 'DB_ROOT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        cp .env.example .env

                        sed -i '/^DB_PASSWORD=/d' .env
                        sed -i '/^DB_ROOT_PASSWORD=/d' .env

                        printf '\\nDB_PASSWORD=%s\\nDB_ROOT_PASSWORD=%s\\n' \
                            "$DB_PASSWORD" "$DB_ROOT_PASSWORD" >> .env

                        sed -i 's/^APP_ENV=.*/APP_ENV=local/' .env
                        sed -i 's/^APP_DEBUG=.*/APP_DEBUG=true/' .env

                        sed -i 's/^DB_CONNECTION=.*/DB_CONNECTION=mysql/' .env
                        sed -i 's/^DB_HOST=.*/DB_HOST=db/' .env
                        sed -i 's/^DB_PORT=.*/DB_PORT=3306/' .env
                        sed -i 's/^DB_DATABASE=.*/DB_DATABASE=laravel/' .env
                        sed -i 's/^DB_USERNAME=.*/DB_USERNAME=laravel_user/' .env
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker compose build app'
            }
        }

        stage('Generate Application Key') {
            steps {
                sh 'docker compose run --rm app php artisan key:generate'
            }
        }

        stage('Deploy Application') {
            steps {
                sh 'docker compose up -d --build'
            }
        }

        stage('Database Migration') {
            steps {
                sh 'docker compose exec -T app php artisan migrate --force'
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    docker compose ps

                    curl --fail --silent --show-error \
                        http://localhost:8000/ \
                        --output /dev/null
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment successful. Application is available at http://localhost:8000'
        }

        failure {
            echo 'Deployment failed. Check the Jenkins console output.'
        }
    }
}
