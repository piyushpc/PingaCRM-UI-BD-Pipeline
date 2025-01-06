pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = "${new Date().format('ddMMMyyyy')}"
        BUILD_DIR = "/home/ubuntu"
        DIST_FILE = ''
        FRONTEND_SERVER = ''
        CREDENTIALS_ID = 'CREDENTIALS_ID'
    }
    


    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'uat', 'prod'],
            description: 'Select the deployment environment: dev, uat, prod'
        )
    }

    stages {
        stage('Debug Environment Variables') {
            steps {
                script {
                    sh 'env | sort'
                }
            }
        }

        stage('Setup AWS Credentials') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
                        env.AWS_ACCESS_KEY_ID = sh(script: 'echo $AWS_ACCESS_KEY_ID', returnStdout: true).trim()
                        env.AWS_SECRET_ACCESS_KEY = sh(script: 'echo $AWS_SECRET_ACCESS_KEY', returnStdout: true).trim()
                    }
                }
            }
        }

        stage('Initialize') {
            steps {
                script {
                    echo "[INFO] Initializing pipeline with parameters: ENVIRONMENT=${params.ENVIRONMENT}, BUILD_DATE=${env.BUILD_DATE}"
                    switch (params.ENVIRONMENT) {
                        case 'dev':
                            env.DIST_FILE = "dist-dev-${env.BUILD_DATE}-new.tar.gz"
                            env.FRONTEND_SERVER = "ec2-3-110-190-110.ap-south-1.compute.amazonaws.com"
                            env.CREDENTIALS_ID = "dev-frontend-ssh-key"
                            break
                        case 'uat':
                            env.DIST_FILE = "dist-uat-${env.BUILD_DATE}-new.tar.gz"
                            env.FRONTEND_SERVER = "crmuat.pingacrm.com"
                            env.CREDENTIALS_ID = "uat-frontend-ssh-key"
                            break
                        default:
                            error "[ERROR] Invalid environment: ${params.ENVIRONMENT}. Use 'dev', 'uat', or 'prod'."
                    }
                    echo "[DEBUG] ENVIRONMENT=${params.ENVIRONMENT}, DIST_FILE=${env.DIST_FILE}, FRONTEND_SERVER=${env.FRONTEND_SERVER}, CREDENTIALS_ID=${env.CREDENTIALS_ID}"
                }
            }
        }

        stage('Clean Old Build Files') {
            steps {
                echo "[INFO] Cleaning up old build files from previous deployments."
                sh "sudo rm -rf /home/ubuntu/pinga/trunk/dist || exit 1"
                echo "[INFO] Old build files removed."
            }
        }

        stage('Copy Environment-Specific Configuration File') {
            steps {
                echo "[INFO] Copying environment-specific configuration file."
                script {
                    def configFilePath = "/home/ubuntu/data.service.ts.${params.ENVIRONMENT}"
                    sh "cp ${configFilePath} /home/ubuntu/pinga/trunk/src/app/service/data.service.ts || exit 1"
                }
                echo "[INFO] Configuration file copied successfully."
            }
        }

        stage('Install Dependencies & Build') {
            steps {
                dir('/home/ubuntu/pinga/trunk') {
                    echo "[INFO] Installing dependencies and preparing build."
                    sh '''
                        rm -rf node_modules package-lock.json
                        npm install --legacy-peer-deps || exit 1
                        npm audit fix || echo "Audit fix failed; ignoring remaining issues."
                        npm audit fix --force || echo "Force audit fix failed."
                        npm run build || exit 1
                        echo "[INFO] Build process completed successfully."
                    '''
                }
            }
        }

        stage('Compress & Upload Build Artifacts') {
            steps {
                dir("${env.BUILD_DIR}/pinga/trunk") {
                    echo "[INFO] Compressing and uploading build artifacts..."
                    script {
                        def DIST_FILE = "dist-${params.ENVIRONMENT}-${env.BUILD_DATE}-new.tar.gz"
                        def TAR_PATH = "${env.BUILD_DIR}/${DIST_FILE}"
                        sh "sudo tar -czvf ${TAR_PATH} dist || exit 1"
                        sh "aws s3 cp ${TAR_PATH} s3://pinga-builds/ || exit 1"
                        echo "[INFO] Build artifact uploaded to S3."
                    }
                }
            }
        }

        stage('Verify Server Availability') {
            steps {
                sshagent(['dev-frontend-ssh-key']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@ec2-3-110-190-110.ap-south-1.compute.amazonaws.com "echo Server is available"'
                }
            }
        }

        stage('Deploy to Server') {
            steps {
                script {
                    if (!CREDENTIALS_ID) {
                        error "CREDENTIALS_ID is not set. Ensure it is defined in the environment section."
                    }
                }
                sshagent(credentials: [CREDENTIALS_ID]) {
                    sh '''
                        ssh -i /home/ubuntu/vkey.pem ubuntu@ec2-3-110-190-110.ap-south-1.compute.amazonaws.com << EOF
                        sudo service apache2 stop || exit 1
                        aws s3 cp s3://pinga-builds/${DIST_FILE} . || exit 1
                        mkdir -p /tmp/${ENVIRONMENT}-dist || exit 1
                        tar -xvf ${DIST_FILE} -C /tmp/${ENVIRONMENT}-dist || exit 1
                        sudo mv /tmp/${ENVIRONMENT}-dist/dist/* /var/www/html/pinga || exit 1
                        sudo chown -R www-data:www-data /var/www/html/pinga || exit 1
                        sudo service apache2 start || exit 1
                        EOF
                    '''
                }
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                echo "Verifying deployment..."
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully."
        }
        failure {
            echo "[ERROR] Pipeline failed. Initiating rollback."
            sshagent(credentials: [CREDENTIALS_ID]) {
                sh '''
                    ssh -i /home/ubuntu/vkey.pem ubuntu@ec2-3-110-190-110.ap-south-1.compute.amazonaws.com << EOF
                    sudo service apache2 stop || exit 1
                    if [ -d /var/www/html/pinga-backup-${BUILD_DATE} ]; then
                        sudo rm -rf /var/www/html/pinga || exit 1
                        sudo mv /var/www/html/pinga-backup-${BUILD_DATE} /var/www/html/pinga || exit 1
                        sudo chown -R www-data:www-data /var/www/html/pinga || exit 1
                        sudo service apache2 start || exit 1
                    else
                        echo "[ERROR] Backup not found. Rollback aborted."
                        exit 1
                    fi
                    EOF
                '''
            }
        }
    }
}
