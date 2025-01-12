pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = "${new Date().format('ddMMMyyyy')}"
        BUILD_DIR = "/home/ubuntu"
        DIST_FILE = "dist-${params.ENVIRONMENT}-${new Date().format('ddMMMyyyy')}-new.tar.gz"
        S3_BUCKET = 'pinga-builds'
        SSH_KEY_PATH = '/var/lib/jenkins/.ssh/vkey.pem'
        SLACK_CHANNEL = "slack-bot-token"
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
                            env.FRONTEND_SERVER = "ec2-3-109-179-70.ap-south-1.compute.amazonaws.com"
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
                                svn update --username ${SVN_USER} --password ${SVN_PASS} ${svnDir}
                                """
                            } else {
                                echo "[INFO] SVN directory does not exist. Performing fresh svn checkout..."
                                sh """
                                svn checkout --username ${SVN_USER} --password ${SVN_PASS} ${svnUrl} ${svnDir}
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
                sh "sudo rm -rf /home/ubuntu/pinga/trunk/dist"
                echo "[INFO] Old build files removed."
            }
        }

        stage('Copy Environment-Specific Configuration File') {
            steps {
                echo "[INFO] Copying environment-specific configuration file."
                script {
                    def configFilePath = "/home/ubuntu/data.service.ts.${params.ENVIRONMENT}"
                    sh "cp ${configFilePath} /home/ubuntu/pinga/trunk/src/app/service/data.service.ts"
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
                        npm install --legacy-peer-deps
                        npm audit fix || echo "Audit fix failed; ignoring remaining issues."
                        npm audit fix --force || echo "Force audit fix failed."
                        npm run build
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
                        sh "sudo tar -czvf ${TAR_PATH} dist"
                        
                        retry(3) {
                            sh """aws s3 cp ${TAR_PATH} s3://pinga-builds/${env.DIST_FILE}"""
                        }
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
                            sudo service apache2 stop
                        "
                    """
                }
            }
        }

        stage('Download and Extract Build from S3') {
    steps {
        sshagent(credentials: [env.CREDENTIALS_ID]) {
            sh """
            echo '[INFO] Downloading the new build from S3...'
            ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                aws s3 cp s3://${S3_BUCKET}/${env.DIST_FILE} /home/ubuntu/${env.DIST_FILE} &&
                echo '[INFO] Build file downloaded successfully.' ||
                (echo '[ERROR] Build file download failed.' && exit 1)
                
                echo '[INFO] Extracting the downloaded build file...'
                tar -xvf /home/ubuntu/${env.DIST_FILE} -C /home/ubuntu/${params.ENVIRONMENT}-dist &&
                echo '[INFO] Build file extracted successfully.' ||
                (echo '[ERROR] Extraction failed.' && exit 1)
            "
            """
        }
    }
}
        stage('Backup Old Build') {
    steps {
        sshagent(credentials: [env.CREDENTIALS_ID]) {
            sh '''
                echo "[INFO] Renaming old dist directory..."
                
                # Set backup directory path with timestamp
                BACKUP_DIR="/var/www/html/pinga-backup-$(date +%d%b%Y%H%M%S)"
                
                # Log the backup directory path
                echo "[DEBUG] Backup directory will be: \$BACKUP_DIR"
                
                # SSH into the server and perform the backup
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} << EOF
                    echo "[INFO] Checking if /var/www/html/pinga exists..."
                    
                    if [ -d /var/www/html/pinga ]; then
                        echo "[INFO] Found /var/www/html/pinga. Proceeding with backup..."
                        
                        # Perform backup and log every step
                        echo "[DEBUG] Running sudo mv command to move old build to backup..."
                        sudo mv /var/www/html/pinga \$BACKUP_DIR
                        
                        if [ \$? -eq 0 ]; then
                            echo "[INFO] Backup completed successfully. Old build moved to \$BACKUP_DIR"
                        else
                            echo "[ERROR] Backup failed during move operation. Exit code: \$?"
                            exit 1
                        fi
                    else
                        echo "[INFO] No existing build found in /var/www/html/pinga. Skipping backup."
                    fi
                EOF
                
                if [ \$? -eq 0 ]; then
                    echo "[INFO] Backup process completed successfully."
                else
                    echo "[ERROR] SSH backup command failed."
                    exit 1
                fi
            '''
        }
    }
}

stage('Prepare Deployment') {
    steps {
        script {
            sh """
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} '
                    echo "[INFO] Renaming old dist directory..."
                    BACKUP_DIR="/var/www/html/pinga-backup-\$(date +%d%b%Y%H%M%S)"
                    if [ -d /var/www/html/pinga ]; then
                        echo "Moving /var/www/html/pinga to \$BACKUP_DIR..."
                        sudo mv /var/www/html/pinga \$BACKUP_DIR
                    else
                        echo "/var/www/html/pinga does not exist, skipping backup."
                    fi
                '
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
                            sudo rm -rf /var/www/html/pinga
                            echo '[INFO] Deploying new build...';
                            sudo mv /home/ubuntu/${params.ENVIRONMENT}-dist/dist/* /var/www/html/pinga
                            echo '[INFO] Updating permissions...';
                            sudo chown -R www-data:www-data /var/www
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

     //   stage('Smoke Tests') {
     //       steps {
     //           script {
      //              echo "[INFO] Running smoke tests..."
       ////             sshagent(credentials: [env.CREDENTIALS_ID]) {
           //             sh """
        //                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
         //               echo \"[INFO] Running smoke tests for application...\"
//
  //                      # Check application health
    //                    curl -sSf https://crmdev.pingacrm.com | grep -q \"<title>Pinga CRM</title>\" || { echo \"[ERROR] Smoke test failed: Content validation failed\"; exit 1; }
//
  //                      # Add more endpoints if needed
    //                    echo \"[INFO] Smoke tests passed successfully.\"
      //                   "
        //                """
          //          }
       //         }
        //    }
     //   }

            stage('Clean Up') {
                steps {
                    echo "[INFO] Cleaning up temporary files."
                    sh "rm -rf /tmp/${params.ENVIRONMENT}-dist"
                }
            }

            stage('Notify Slack') {
    steps {
        script {
            def message = "Deployment of ${params.ENVIRONMENT} build completed successfully!"
            slackSend(
                channel: 'jenkins', // Replace with your Slack channel
                color: 'good',
                message: message,
                tokenCredentialId: 'slack-bot-token' // Replace with the ID of your Slack credential
            )
        }
    }
}
    }
    
    post {
        failure {
            slackSend(channel: env.SLACK_CHANNEL, color: 'danger', message: "Deployment failed for ${params.ENVIRONMENT}. Please investigate.")
        }
    }
}
