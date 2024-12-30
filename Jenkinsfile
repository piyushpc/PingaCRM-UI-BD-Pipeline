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
                            env.FRONTEND_SERVER = "ec2-15-207-107-0.ap-south-1.compute.amazonaws.com"
                            env.CREDENTIALS_ID = "dev-frontend-ssh-key"
                            break
                        case 'uat':
                            env.DIST_FILE = "dist-uat-${env.BUILD_DATE}-new.tar.gz"
                            env.FRONTEND_SERVER = "ec2-13-235-77-253.ap-south-1.compute.amazonaws.com"
                            env.CREDENTIALS_ID = "uat-frontend-ssh-key"
                            break
                        case 'prod':
                            env.DIST_FILE = "dist-prod-${env.BUILD_DATE}-new.tar.gz"
                            env.FRONTEND_SERVER = "crmprod.pingacrm.com"
                            env.CREDENTIALS_ID = "prod-frontend-ssh-key"
                            break
                        default:
                            error "[ERROR] Invalid environment: ${params.ENVIRONMENT}. Use 'dev', 'uat', or 'prod'."
                    }
                    echo "[INFO] Configuration set: DIST_FILE=${env.DIST_FILE}, FRONTEND_SERVER=${env.FRONTEND_SERVER}"
                }
            }
        }

        stage('Backup Current Code') {
            steps {
                echo "[INFO] Backing up current deployment at /home/ubuntu/pinga"
                sh "sudo cp -R /home/ubuntu/pinga /home/ubuntu/pinga-${env.BUILD_DATE}-backup || exit 1"
                echo "[INFO] Backup completed successfully."
            }
        }

        stage('Clean Old Build Files') {
            steps {
                echo "[INFO] Cleaning up old build files from previous deployments."
                sh "sudo rm -rf /home/ubuntu/pinga/trunk/dist || exit 1"
                echo "[INFO] Old build files removed."
            }
        }

        stage('Fetch Latest Code from SVN') {
            steps {
                script {
                    echo "[INFO] Fetching latest code from SVN repository."
                    def svnUrl = "https://extsvn.pingacrm.com/svn/pingacrm-frontend-new/trunk"

                    withCredentials([usernamePassword(credentialsId: 'svn-credentials-id', 
                                                      usernameVariable: 'SVN_USER', 
                                                      passwordVariable: 'SVN_PASS')]) {
                        sh """
                        svn checkout --username ${SVN_USER} --password ${SVN_PASS} ${svnUrl} /home/ubuntu/pinga/trunk || exit 1
                        """
                    }

                    echo "[INFO] SVN checkout/update completed successfully."
                }
            }
        }

        stage('Install Dependencies & Build') {
    steps {
        dir('/home/ubuntu/pinga/trunk') {
            echo "[INFO] Installing dependencies and preparing build."
            sh '''

                script {
    def configFile
    switch (params.ENVIRONMENT) {
        case 'dev':
            configFile = '/home/ubuntu/data.service.ts.dev'
            break
        case 'uat':
            configFile = '/home/ubuntu/data.service.ts.uat'
            break
        case 'prod':
            configFile = '/home/ubuntu/data.service.ts.prod'
            break
    }
    sh "cp ${configFile} /home/ubuntu/pinga/trunk/src/app/service/data.service.ts || exit 1"
    echo "[INFO] Copied configuration file: ${configFile}"
}

                sh 'echo "Cleaning up node_modules and package-lock.json"'
                sh 'ls -l node_modules'
                rm -rf node_modules package-lock.json || exit 1
                echo "[INFO] Removed existing dependencies."
                npm install --legacy-peer-deps --no-audit --no-fund || exit 1

                echo "[INFO] Dependencies installed successfully."
                while true; do
                    export NODE_OPTIONS=--max_old_space_size=4096 && npm run build -- --progress=true --no-sourcemap
                    echo "[INFO] Build still in progress..." && sleep 30
                done
                echo "[INFO] Build process completed successfully."
                ls -la dist || exit 1
                sudo tar -czvf ${BUILD_DIR}/${DIST_FILE} dist || exit 1
                echo "[INFO] Build artifacts compressed into ${BUILD_DIR}/${DIST_FILE}."
            '''
        }
    }
}


        stage('Upload Build Artifact to S3') {
            steps {
                echo "[INFO] Uploading build artifact to S3 bucket."
                sh "aws s3 cp ${BUILD_DIR}/${DIST_FILE} s3://pinga-builds/ || exit 1"
                echo "[INFO] Build artifact uploaded successfully to s3://pinga-builds/${DIST_FILE}."
            }
        }

        stage('Deploy to Server') {
            steps {
                echo "FRONTEND_SERVER=${env.FRONTEND_SERVER}"
                echo "[INFO] Deploying build artifact to server: ${FRONTEND_SERVER}"
                sh '''
                    sudo -u jenkins ssh -i /home/ubuntu/vkey.pem ubuntu@${FRONTEND_SERVER} << EOF
                    echo "[INFO] Stopping Apache server."
                    sudo service apache2 stop || exit 1
                    echo "[INFO] Apache server stopped."

                    echo "[INFO] Downloading build artifact from S3."
                    aws s3 cp s3://pinga-builds/${DIST_FILE} . || exit 1
                    echo "[INFO] Build artifact downloaded successfully."

                    if [ -d /var/www/html/pinga ]; then
                        echo "[INFO] Backing up existing deployment."
                        sudo mv /var/www/html/pinga /var/www/html/pinga-backup-${BUILD_DATE} || exit 1
                        echo "[INFO] Backup of existing deployment completed."
                    fi

                    echo "[INFO] Preparing temporary directory for deployment."
                    mkdir -p /tmp/${ENVIRONMENT}-dist || exit 1
                    tar -xvf ${DIST_FILE} -C /tmp/${ENVIRONMENT}-dist || exit 1
                    echo "[INFO] Build artifact extracted successfully."

                    echo "[INFO] Deploying new build to web server."
                    sudo mv /tmp/${ENVIRONMENT}-dist/dist/* /var/www/html/pinga || exit 1
                    sudo chown -R www-data:www-data /var/www/html/pinga || exit 1
                    echo "[INFO] New build deployed successfully."

                    echo "[INFO] Starting Apache server."
                    sudo service apache2 start || exit 1
                    echo "[INFO] Apache server started."
                    EOF
                '''
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                script {
                    echo "[INFO] Verifying deployment by checking application health."
                    def testCommand = "curl --fail https://${FRONTEND_SERVER} || exit 1"
                    sh '''
                        sudo -u jenkins ssh -i /home/ubuntu/vkey.pem ubuntu@${FRONTEND_SERVER} ${testCommand}
                    '''
                    echo "[INFO] Deployment verification successful. Application is accessible."
                }
            }
        }
    }

    post {
        success {
            echo "[INFO] Pipeline executed successfully."
        }
        failure {
            echo "[ERROR] Pipeline failed. Initiating rollback."
            sh '''
                sudo -u jenkins ssh -i /home/ubuntu/vkey.pem ubuntu@${FRONTEND_SERVER} << EOF
                if [ -d /var/www/html/pinga-backup-${BUILD_DATE} ]; then
                    echo "[INFO] Restoring previous deployment."
                    sudo rm -rf /var/www/html/pinga || exit 1
                    sudo mv /var/www/html/pinga-backup-${BUILD_DATE} /var/www/html/pinga || exit 1
                    sudo chown -R www-data:www-data /var/www/html/pinga || exit 1
                    echo "[INFO] Rollback to previous deployment completed."

                    echo "[INFO] Restarting Apache server."
                    sudo service apache2 start || exit 1
                    echo "[INFO] Apache server restarted."
                else
                    echo "[ERROR] No backup found for rollback. Manual intervention required."; exit 1
                fi
                EOF
            '''
        }
    }
}
