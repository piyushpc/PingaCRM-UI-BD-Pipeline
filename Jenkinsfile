pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = "${new Date().format('ddMMMyyyy')}"
        BUILD_DIR = "/home/ubuntu"
        DIST_FILE = "dist-dev-${sh(script: 'date +\"%d%b%Y\"', returnStdout: true).trim()}-new.tar.gz"
        FRONTEND_SERVER = 'ec2-65-2-170-67.ap-south-1.compute.amazonaws.com'
        CREDENTIALS_ID = 'CREDENTIALS_ID'
        S3_BUCKET = 'pinga-builds'
        SSH_KEY_PATH = '/home/ubuntu/vkey.pem'
        SLACK_CHANNEL = "jenkins"
    }

    parameters {
        booleanParam(name: 'UPDATE_SVN', defaultValue: false, description: 'Set to true to fetch the latest code from SVN')
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
                            env.FRONTEND_SERVER = "ec2-65-2-170-67.ap-south-1.compute.amazonaws.com"
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

        stage('Backup Current Code') {
            steps {
                script {
                    def backupPath = "/home/ubuntu/pinga-${env.BUILD_DATE}-backup"
                    echo "[INFO] Backing up current deployment at /home/ubuntu/pinga to ${backupPath}"
                    sh """
                        if [ ! -d /home/ubuntu/pinga ]; then
                            echo "[ERROR] Source directory /home/ubuntu/pinga does not exist!"
                            exit 1
                        fi
                        sudo cp -R /home/ubuntu/pinga ${backupPath}
                        if [ ! -d ${backupPath} ]; then
                            echo "[ERROR] Backup failed!"
                            exit 1
                        fi
                    """
                    echo "[INFO] Backup completed successfully at ${backupPath}."
                }
            }
        }

        stage('Handle SVN Checkout/Update') {
    steps {
        script {
            if (params.UPDATE_SVN) {
                echo "[INFO] UPDATE_SVN is enabled. Checking SVN directory..."
                
                // Define SVN URL and local directory
                def svnUrl = "https://extsvn.pingacrm.com/svn/pingacrm-frontend-new/trunk"
                def svnDir = "/home/ubuntu/pinga/trunk"

                withCredentials([usernamePassword(credentialsId: 'svn-credentials-id', 
                                                  usernameVariable: 'SVN_USER', 
                                                  passwordVariable: 'SVN_PASS')]) {
                    // Check if the SVN directory exists
                    def dirExists = sh(script: "if [ -d ${svnDir} ]; then echo exists; else echo not_exists; fi", returnStdout: true).trim()
                    
                    if (dirExists == "exists") {
                        echo "[INFO] SVN directory exists. Performing svn update..."
                        sh """
                        svn update --username ${SVN_USER} --password ${SVN_PASS} ${svnDir} || exit 1
                        """
                    } else {
                        echo "[INFO] SVN directory does not exist. Performing fresh svn checkout..."
                        sh """
                        svn checkout --username ${SVN_USER} --password ${SVN_PASS} ${svnUrl} ${svnDir} || exit 1
                        """
                    }
                }

                echo "[INFO] SVN operation completed successfully."
            } else {
                echo "[INFO] UPDATE_SVN is disabled. Skipping SVN operations."
            }
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
                        sh """aws s3 cp ${TAR_PATH} s3://pinga-builds/${env.DIST_FILE} || { echo '[ERROR] S3 upload failed'; exit 1; }"""
                        echo "[INFO] Build artifact uploaded to S3."
                    }
                }
            }
        }

        stage('Verify Server Availability') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "echo [INFO] Server is reachable."
                    """
                    echo "[INFO] Deployment process started..."
                }
            }
        }

        stage('Stop Apache') {
    steps {
        sshagent(credentials: [env.CREDENTIALS_ID]) {
            sh """
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                    echo '[INFO] Stopping Apache...';
                    sudo service apache2 stop || { echo '[ERROR] Failed to stop Apache'; exit 1; }
                "
            """
        }
    }
}
        
        stage('Download Build from S3') {
    steps {
        sshagent(credentials: [env.CREDENTIALS_ID]) {
            sh """
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} << 'EOF'
                    echo '[INFO] Downloading the new build from S3...'
                            aws s3 cp s3://${S3_BUCKET}/${env.DIST_FILE} . || { echo '[ERROR] S3 download failed'; exit 1; }
                        EOF
            """
        }
    }
        }

        stage('Backup Old Build') {
    steps {
        sshagent(credentials: [env.CREDENTIALS_ID]) {
            sh """
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                    echo '[INFO] Renaming old dist directory...';
                    if [ -d /var/www/html/pinga ]; then
                        BACKUP_DIR='/var/www/html/pinga-backup-\$(date +%d%b%Y%H%M%S)'
                        sudo mv /var/www/html/pinga \$BACKUP_DIR || { echo '[ERROR] Backup failed'; exit 1; }
                    fi
                "
            """
        }
    }
}




        stage('Prepare Deployment') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            echo '[INFO] Ensuring deployment directory exists...';
                            mkdir -p /tmp/${params.ENVIRONMENT}-dist || { echo '[ERROR] Failed to create directory'; exit 1; }

                            echo '[INFO] Unzipping the new build...';
                            tar -xvf ${env.DIST_FILE} -C /tmp/${params.ENVIRONMENT}-dist || { echo '[ERROR] Unzipping failed'; exit 1; }
                        "
                    """
                }
            }
        }

        stage('Deploy New Build') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            echo '[INFO] Removing old deployment...';
                            sudo rm -rf /var/www/html/pinga || { echo '[ERROR] Failed to remove old deployment'; exit 1; }

                            echo '[INFO] Deploying new build...';
                            sudo mv /tmp/${params.ENVIRONMENT}-dist/dist/* /var/www/html/pinga || { echo '[ERROR] Deployment failed'; exit 1; }

                            echo '[INFO] Updating permissions...';
                            sudo chown -R www-data:www-data /var/www || { echo '[ERROR] Failed to update permissions'; exit 1; }
                        "
                    """
                }
            }
        }

        stage('Start Apache') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            echo '[INFO] Starting Apache...';
                            sudo service apache2 start || { echo '[ERROR] Failed to start Apache'; exit 1; }
                        "
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            echo '[INFO] Cleaning up temporary directories...';
                            sudo rm -rf /tmp/${params.ENVIRONMENT}-dist || { echo '[ERROR] Failed to clean up temporary directories'; exit 1; }
                        "
                    """
                }
            }
        }

        stage('Finalize') {
            steps {
                script {
                    if (currentBuild.result == 'FAILURE') {
                        echo "[ERROR] Deployment process encountered errors."
                    } else {
                        echo "[INFO] Deployment process completed successfully."
                    }
                }
            }
        }

        // Correct placement of the Smoke Tests stage within the 'stages' block
        stage('Smoke Tests') {
            steps {
                script {
                    echo "[INFO] Running smoke tests..."
                    sshagent(credentials: [env.CREDENTIALS_ID]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                                echo '[INFO] Running smoke tests for application...';
                                curl -sSf https://crmdev.pingacrm.com | grep -q '<title>Pinga CRM</title>' || { echo '[ERROR] Smoke test failed'; exit 1; }
                                echo '[INFO] Smoke tests passed successfully.';
                            "
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo "[INFO] Deployment completed successfully. Sending success notification..."
                emailext(
                    subject: "PingaCRM Deployment Successful",
                    body: "Deployment for ${params.ENVIRONMENT} completed successfully.",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                slackSend(
                color: 'good',
                message: "PingaCRM Deployment Successful for ${params.ENVIRONMENT} :white_check_mark:",
                tokenCredentialId: 'slack-bot-token'  // Replace with your actual credential ID
            )
            }
        }

    //    failure {
      //      script {
        //        echo "[ERROR] Deployment failed. Sending failure notification..."
          //      emailext(
            //        subject: "PingaCRM Deployment Failed",
              //      body: "Deployment for ${params.ENVIRONMENT} failed. Please check Jenkins logs.",
                //    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
              //  )
              //  slackSend(
                //    color: 'danger',
                //    message: "PingaCRM Deployment Failed for ${params.ENVIRONMENT}. :x:"
             //   )
   //         }
    //    }
 //   }
//}

        failure {
            script {
                echo "[ERROR] Deployment failed. Initiating rollback and sending failure notification..."
                // Rollback logic
                sshagent(credentials: [CREDENTIALS_ID]) {
                    sh '''
                        ssh -i /home/ubuntu/vkey.pem ubuntu@${env.FRONTEND_SERVER} << EOF
                        sudo service apache2 stop || exit 1
                        if [ -d /var/www/html/pinga-backup-${env.BUILD_DATE} ]; then
                            sudo rm -rf /var/www/html/pinga || exit 1
                            sudo mv /var/www/html/pinga-backup-${env.BUILD_DATE} /var/www/html/pinga || exit 1
                        fi
                        sudo service apache2 start || exit 1
                        EOF
                    '''
                }
            }
        }
    }
}
