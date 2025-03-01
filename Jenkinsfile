pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        EC2_HOST = '13.127.5.70'
        EC2_USER = 'ubuntu'
        SSH_CRED_ID = 'ec2-ssh-key'
        DEPLOY_DIR = '/var/www/laravel-react-forum'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/aqeelgardezi/testing.git'
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Frontend - ReactJS') {
                    steps {
                        sh 'npm install'
                    }
                }
                stage('Backend - Laravel') {
                    steps {
                        sh 'composer install --no-dev --optimize-autoloader'
                    }
                }
            }
        }

        stage('Dependency Checks') {
            parallel {
                stage('OWASP Dependency-Check') {
                    steps {
                        dependencyCheck additionalArguments: '--format HTML --format XML --scan .', odcInstallation: 'Dependency-Check'
                        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    }
                }
                stage('NPM Audit') {
                    steps {
                        sh 'npm audit --json > npm-audit-report.json || true'
                        archiveArtifacts artifacts: 'npm-audit-report.json', allowEmptyArchive: true
                    }
                }
            }
        }

        stage('Code Coverage') {
            parallel {
                stage('Frontend - Jest') {
                    steps {
                        sh 'npm test -- --coverage --ci --reporters=default --reporters=jest-junit'
                        junit 'junit.xml'
                        publishHTML(target: [
                            reportDir: 'coverage/lcov-report',
                            reportFiles: 'index.html',
                            reportName: 'Frontend Coverage'
                        ])
                    }
                }
                stage('Backend - PHPUnit') {
                    steps {
                        sh 'vendor/bin/phpunit --coverage-html coverage --log-junit phpunit-report.xml'
                        junit 'phpunit-report.xml'
                        publishHTML(target: [
                            reportDir: 'coverage',
                            reportFiles: 'index.html',
                            reportName: 'Backend Coverage'
                        ])
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run production'
                sh 'tar -czf app.tar.gz --exclude="node_modules" --exclude="vendor" .'
                archiveArtifacts artifacts: 'app.tar.gz', fingerprint: true
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent([SSH_CRED_ID]) {
                    sh """
                    scp -o StrictHostKeyChecking=no app.tar.gz ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/
                    ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << 'EOF'
                        cd ${DEPLOY_DIR}
                        tar -xzf app.tar.gz
                        touch database/database.sqlite
                        chmod 664 database/database.sqlite
                        composer install --no-dev --optimize-autoloader
                        php artisan key:generate  # Added to ensure APP_KEY is set
                        php artisan migrate --force
                        php artisan config:cache
                        php artisan route:cache
                        sudo chown -R www-data:www-data ${DEPLOY_DIR}
                        sudo chmod -R 775 ${DEPLOY_DIR}/storage ${DEPLOY_DIR}/bootstrap/cache
                        sudo systemctl restart apache2
                    EOF
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Deployment to Ubuntu EC2 successful!'
        }
        failure {
            echo 'Pipeline failed. Check logs.'
        }
    }
}