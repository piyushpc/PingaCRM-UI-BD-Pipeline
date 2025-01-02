pipeline {
    agent any
    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = new Date().format('ddMMMyyyy', TimeZone.getTimeZone('UTC'))
    }
    stages {
        stage('Checkout Code') {
            steps {
                echo '[INFO] Checking out code from GitHub.'
                checkout scm
            }
        }

        stage('Debug Environment Variables') {
            steps {
                script {
                    echo '[INFO] Debugging environment variables.'
                    sh '''
                        env | sort
                    '''
                }
            }
        }

        stage('Setup AWS Credentials') {
            steps {
                script {
                    echo '[INFO] Setting up AWS credentials.'
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh '''
                            echo "AWS_ACCESS_KEY_ID = ****"
                            echo "AWS_SECRET_ACCESS_KEY = ****"
                        '''
                    }
                }
            }
        }

        stage('Test AWS Access') {
            steps {
                script {
                    echo '[INFO] Testing AWS S3 access.'
                    sh '''
                        aws s3 ls s3://pinga-builds || exit 1
                    '''
                }
            }
        }

        stage('Initialize') {
            steps {
                script {
                    echo '[INFO] Initializing pipeline.'
                    // Add validation for essential files or environment variables
                    if (!fileExists('config/settings.json')) {
                        error '[ERROR] Initialization failed: Missing config/settings.json file.'
                    }
                    echo '[INFO] Initialization successful.'
                }
            }
        }

        stage('Clean Old Build Files') {
            steps {
                echo '[INFO] Cleaning old build files.'
                sh '''
                    rm -rf build/*
                '''
            }
        }

        stage('Copy Environment-Specific Configuration File') {
            steps {
                echo '[INFO] Copying environment-specific configuration file.'
                sh '''
                    cp config/settings.${ENVIRONMENT}.json build/settings.json
                '''
            }
        }

        stage('Install Dependencies & Build') {
            steps {
                echo '[INFO] Installing dependencies and building the project.'
                sh '''
                    npm install
                    npm run build
                '''
            }
        }

        stage('Compress & Upload Build Artifacts') {
            steps {
                echo '[INFO] Compressing and uploading build artifacts.'
                sh '''
                    tar -czvf build-${BUILD_DATE}.tar.gz build/
                    aws s3 cp build-${BUILD_DATE}.tar.gz s3://pinga-builds/
                '''
            }
        }

        stage('Deploy to Server') {
            steps {
                echo '[INFO] Deploying to server.'
                sh '''
                    ssh -i /path/to/private/key ubuntu@${SERVER_IP} << EOF
                        sudo systemctl stop myapp
                        rm -rf /var/www/html/*
                        tar -xvf build-${BUILD_DATE}.tar.gz -C /var/www/html/
                        sudo systemctl start myapp
                    EOF
                '''
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                echo '[INFO] Performing post-deployment verification.'
                sh '''
                    curl -f http://your-server-url/health || exit 1
                '''
            }
        }
    }
    post {
        success {
            echo '[SUCCESS] Pipeline completed successfully.'
        }
        failure {
            echo '[ERROR] Pipeline failed. Please review the logs.'
        }
    }
}
