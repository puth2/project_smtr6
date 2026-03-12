pipeline {
    agent any
    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/puth2/project_smtr6.git'
            }
        }
        stage('Build & Install') {
            steps {
                script {
                    docker.image('php:8.4-cli').inside('-u root') {
                        sh '''
                            apt-get update && apt-get install -y zip unzip curl libzip-dev libpng-dev libonig-dev
                            docker-php-ext-install pdo_mysql mbstring zip

                            curl -sS https://getcomposer.org/installer -o composer-setup.php
                            php composer-setup.php --install-dir=/usr/local/bin --filename=composer
                            rm composer-setup.php

                            composer install --no-interaction --prefer-dist --optimize-autoloader

                            if [ ! -f .env ]; then cp .env.example .env; fi
                            php artisan key:generate
                        '''
                    }
                }
            }
        }
        stage('Deploy to Simulation Server') {
            steps {
                sshagent(credentials: ['ssh-prod']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no root@172.27.12.115 "mkdir -p /root/prod_server"

                        scp -o StrictHostKeyChecking=no -r ./* root@172.27.12.115:/root/prod_server/

                        ssh -o StrictHostKeyChecking=no root@172.27.12.115 "
                            cd /root/prod_server

                            docker rm -f laravel-online || true

                            docker run -d --name laravel-online \
                                -p 8081:8000 \
                                -v /root/prod_server:/var/www/html \
                                -w /var/www/html \
                                php:8.4-cli php artisan serve --host=0.0.0.0

                            sleep 2
                            docker exec laravel-online bash -c 'if [ ! -f .env ]; then cp .env.example .env; fi'
                            docker exec laravel-online php artisan key:generate --force
                            docker exec -u root laravel-online chmod -R 777 storage bootstrap/cache

                            echo '✅ Laravel project_smtr6 berjalan di port 8081'
                        "
                        echo "✅ Deploy selesai! Akses di http://172.27.12.115:8081"
                    '''
                }
            }
        }
    }
    post {
        success {
            echo '🚀 Pipeline sukses! project_smtr6 live di http://172.27.12.115:8081'
        }
        failure {
            echo '❌ Pipeline gagal. Cek log di atas untuk detail error.'
        }
    }
}