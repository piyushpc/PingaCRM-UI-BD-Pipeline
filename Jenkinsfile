pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = "${new Date().format('ddMMMyyyy')}"
        BUILD_DIR = "/home/ubuntu"
        DIST_FILE = ''
        FRONTEND_SERVER = ''
        CREDENTIALS_ID = ''
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
                    echo "[INFO] Debugging environment variables."
                    sh 'env | sort'
                }
            }
        }

        stage('Setup AWS Credentials') {
            steps {
                script {
                    echo "[INFO] Setting up AWS credentials."
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
                        env.AWS_ACCESS_KEY_ID = sh(script: 'echo $AWS_ACCESS_KEY_ID', returnStdout: true).trim()
                        env.AWS_SECRET_ACCESS_KEY = sh(script: 'echo $AWS_SECRET_ACCESS_KEY', returnStdout: true).trim()
                    }
                }
            }
        }

        stage('Test AWS Access') {
            steps {
                echo "[INFO] Testing AWS S3 access."
                sh 'aws s3 ls s3://pinga-builds || exit 1'
            }
        }

        stage('Initialize') {
            steps {
                script {
                    echo "[INFO] Initializing pipeline."
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
                        case 'prod':
                            env.DIST_FILE = "dist-prod-${env.BUILD_DATE}-new.tar.gz"
                            env.FRONTEND_SERVER = "crmprod.pingacrm.com"
                            env.CREDENTIALS_ID = "prod-frontend-ssh-key"
                            break
                        default:
                            error "[ERROR] Invalid environment selected: ${params.ENVIRONMENT}"
                    }
                    if (!env.DIST_FILE || !env.FRONTEND_SERVER || !env.CREDENTIALS_ID) {
                        error "[ERROR] Initialization failed: Missing configuration."
                    }
                    echo "[INFO] Initialization successful: DIST_FILE=${env.DIST_FILE}, FRONTEND_SERVER=${env.FRONTEND_SERVER}"
                }
            }
        }

        stage('Clean Old Build Files') {
            steps {
                echo "[INFO] Cleaning old build files."
                sh '''
                    if [ -d "/home/ubuntu/pinga/trunk/dist" ]; then
                        sudo rm -rf /home/ubuntu/pinga/trunk/dist || exit 1
                        echo "[INFO] Old build files removed."
                    else
                        echo "[INFO] No old build files found."
                    fi
                '''
            }
        }

        stage('Copy Environment-Specific Configuration File') {
            steps {
                script {
                    def configFilePath = "/home/ubuntu/data.service.ts.${params.ENVIRONMENT}"
                    echo "[INFO] Copying environment-specific configuration file from ${configFilePath}"
                    sh "cp ${configFilePath} /home/ubuntu/pinga/trunk/src/app/service/data.service.ts || exit 1"
                }
            }
        }

        stage('Install Dependencies & Build') {
            steps {
                dir('/home/ubuntu/pinga/trunk') {
                    echo "[INFO] Installing dependencies and building project."
                    sh '''
                        rm -rf node_modules package-lock.json || echo "No dependencies to clean."
                        npm install --legacy-peer-deps || exit 1
                        npm audit fix || echo "[WARNING] Some vulnerabilities remain."
                        npm run build || exit 1
                    '''
                }
            }
        }

        stage('Compress & Upload Build Artifacts') {
            steps {
                dir('/home/ubuntu/pinga/trunk') {
                    script {
                        def TAR_PATH = "${env.BUILD_DIR}/${env.DIST_FILE}"
                        echo "[INFO] Compressing build artifacts to ${TAR_PATH}."
                        sh "sudo tar -czvf ${TAR_PATH} dist || exit 1"

                        echo "[INFO] Uploading build artifact to S3."
                        sh "aws s3 cp ${TAR_PATH} s3://pinga-builds/ || exit 1"
                    }
                }
            }
        }

        stage('Deploy to Server') {
            steps {
                echo "[INFO] Deploying to server: ${env.FRONTEND_SERVER}"
                withCredentials([sshUserPrivateKey(credentialsId: "${env.CREDENTIALS_ID}", keyFileVariable: 'SSH_KEY_PATH')]) {
                    sh '''
                        ssh -i ${SSH_KEY_PATH} ubuntu@${FRONTEND_SERVER} << EOF
                        sudo service apache2 stop || exit 1
                        aws s3 cp s3://pinga-builds/${DIST_FILE} . || exit 1
                        mkdir -p /tmp/deploy-dist || exit 1
                        tar -xvf ${DIST_FILE} -C /tmp/deploy-dist || exit 1
                        sudo mv /tmp/deploy-dist/dist/* /var/www/html/pinga || exit 1
                        sudo chown -R www-data:www-data /var/www/html/pinga || exit 1
                        sudo service apache2 start || exit 1
                        EOF
                    '''
                }
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                echo "[INFO] Verifying deployment."
                sh '''
                    curl --fail https://${FRONTEND_SERVER} || exit 1
                '''
                echo "[INFO] Deployment verification successful."
            }
        }
    }

    post {
        success {
            echo "[INFO] Pipeline completed successfully."
        }
        failure {
            echo "[ERROR] Pipeline failed. Please review the logs."
        }
    }
}
