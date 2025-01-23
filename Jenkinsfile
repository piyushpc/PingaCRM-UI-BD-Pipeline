
pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = sh(script: 'date +%d%b%Y', returnStdout: true).trim()
        BUILD_DIR = "/home/ubuntu"
        DIST_FILE = "dist-${params.ENVIRONMENT}-${new Date().format('ddMMMyyyy')}-new.tar.gz"
        S3_BUCKET = 'pinga-builds'
        SSH_KEY_PATH = '/var/lib/jenkins/.ssh/vkey.pem'
        SLACK_CHANNEL = "slack-bot-token"
        BUILD_SERVER = "ec2-3-110-193-16.ap-south-1.compute.amazonaws.com"
        SSH_CREDENTIALS_ID = "build-server-ssh-credentials-id"
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
                    sshagent([env.SSH_CREDENTIALS_ID]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.BUILD_SERVER} 'env | sort'
                        """
                    }
                }
            }
        }

        stage('Setup AWS Credentials') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
                        sshagent([env.SSH_CREDENTIALS_ID]) {
                            sh """
                                ssh -o StrictHostKeyChecking=no ubuntu@${env.BUILD_SERVER} << EOF
                                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                                EOF
                            """
                        }
                    }
                }
            }
        }


        stage('Initialize') {
    steps {
        script {
            echo "[INFO] Initializing pipeline with parameters: ENVIRONMENT=${params.ENVIRONMENT}, BUILD_DATE=${env.BUILD_DATE}"
            
            // SSH into the build server before proceeding with the environment setup
            echo "[INFO] Connecting to the build server ${env.BUILD_SERVER}..."
            sshagent(credentials: [env.CREDENTIALS_ID]) {
                sh '''
                    echo "[INFO] SSH connection established successfully!"
                    # Any additional SSH commands you may want to run before continuing
                '''
            }
            
            // Environment-specific setup
            switch (params.ENVIRONMENT) {
                case 'dev':
                    env.DIST_FILE = "dist-dev-${env.BUILD_DATE}-new.tar.gz"
                    env.FRONTEND_SERVER = "ec2-3-109-179-70.ap-south-1.compute.amazonaws.com"
                    env.CREDENTIALS_ID = "dev-frontend-ssh-key"
                    break
                case 'uat':
                    env.DIST_FILE = "dist-uat-${env.BUILD_DATE}-new.tar.gz"
                    env.FRONTEND_SERVER = "ec2-65-1-130-96.ap-south-1.compute.amazonaws.com"
                    env.CREDENTIALS_ID = "uat-frontend-ssh-key"
                    break
                case 'prod':
                    env.DIST_FILE = "dist-prod-${env.BUILD_DATE}-new.tar.gz"
                    env.FRONTEND_SERVER = "prod.pingacrm.com"
                    env.CREDENTIALS_ID = "prod-frontend-ssh-key"
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
                    sshagent([env.SSH_CREDENTIALS_ID]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.BUILD_SERVER} << EOF
                                BACKUP_DIR="${env.BUILD_DIR}/pinga-backup-${env.BUILD_DATE}"
                                if [ -d ${env.BUILD_DIR}/pinga ]; then
                                    echo "[INFO] Backing up current deployment..."
                                    cp -R ${env.BUILD_DIR}/pinga ${BACKUP_DIR}
                                    echo "[INFO] Backup completed at ${BACKUP_DIR}"
                                else
                                    echo "[WARNING] No existing deployment found; skipping backup."
                                fi
                            EOF
                        """
                    }
                }
            }
        }

        stage('Handle SVN Checkout/Update') {
            steps {
                script {
                    if (params.UPDATE_SVN) {
                        sshagent([env.SSH_CREDENTIALS_ID]) {
                            withCredentials([usernamePassword(credentialsId: 'svn-credentials-id', 
                                                              usernameVariable: 'SVN_USER', 
                                                              passwordVariable: 'SVN_PASS')]) {
                                sh """
                                    ssh -o StrictHostKeyChecking=no ubuntu@${env.BUILD_SERVER} << EOF
                                        echo "[INFO] Performing SVN checkout..."
                                        svn checkout --username $SVN_USER --password $SVN_PASS \
                                        https://extsvn.pingacrm.com/svn/pingacrm-frontend-new/trunk \
                                        ${env.BUILD_DIR}/pinga/trunk
                                    EOF
                                """
                            }
                        }
                    } else {
                        echo "[INFO] UPDATE_SVN is disabled. Skipping SVN operations."
                    }
                }
            }
        }

        stage('Install Dependencies & Build') {
            steps {
                script {
                    sshagent([env.SSH_CREDENTIALS_ID]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.BUILD_SERVER} << EOF
                                cd ${env.BUILD_DIR}/pinga/trunk
                                rm -rf dist node_modules package-lock.json
                                npm install --legacy-peer-deps
                                npm run build
                            EOF
                        """
                    }
                }
            }
        }

        stage('Compress & Upload Build Artifacts') {
            steps {
                script {
                    sshagent([env.SSH_CREDENTIALS_ID]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.BUILD_SERVER} << EOF
                                cd ${env.BUILD_DIR}/pinga/trunk
                                tar -czvf ${env.DIST_FILE} dist
                                aws s3 cp ${env.BUILD_DIR}/${env.DIST_FILE} s3://${env.S3_BUCKET}/${env.DIST_FILE}
                            EOF
                        """
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

                echo '[INFO] Ensuring target directory exists...'
                mkdir -p /home/ubuntu/${params.ENVIRONMENT}-dist ||
                (echo '[ERROR] Failed to create target directory.' && exit 1)

                echo '[INFO] Extracting the downloaded build file...'
                sudo tar -xvf /home/ubuntu/${env.DIST_FILE} -C /home/ubuntu/${params.ENVIRONMENT}-dist &&
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
            sh """
                ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/vkey.pem ubuntu@ec2-3-109-179-70.ap-south-1.compute.amazonaws.com '
                    echo "[INFO] Renaming old dist directory...";
                    if [ -d /var/www/html/pinga ]; then
                        BACKUP_DIR="/home/ubuntu/backup-${params.ENVIRONMENT}-\$(date +%d%b%Y_%H%M%S)"
                        echo "[INFO] Creating unique backup directory: \$BACKUP_DIR";
                        mkdir -p \$BACKUP_DIR || { echo "[ERROR] Failed to create backup directory"; exit 1; }
                        sudo mv /var/www/html/pinga \$BACKUP_DIR || { echo "[ERROR] Backup failed"; exit 1; }
                        echo "[INFO] Backup completed successfully at \$BACKUP_DIR.";
                    else
                        echo "[INFO] No directory to backup.";
                    fi
                '
            """
        }
    }
}


        stage('Prepare Deployment') {
    steps {
        script {
            try {
                sh """
                    ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} '
                        # Use Bash explicitly
                        if [ -x /bin/bash ]; then
                            exec /bin/bash
                        fi

                        # Enable debugging and strict error handling
                        set -e
                        set -o pipefail

                        echo "[INFO] Starting deployment preparation..."
                        BACKUP_DIR="/home/ubuntu/pinga-backup-\$(date +%d%b%Y%H%M%S)"
                        echo "[DEBUG] Calculated backup directory: \$BACKUP_DIR"

                        if [ -d /var/www/html/pinga ]; then
                            echo "[INFO] Found existing /var/www/html/pinga directory."
                            echo "[INFO] Moving it to backup directory..."
                            sudo mv /var/www/html/pinga \$BACKUP_DIR
                            echo "[INFO] Successfully moved /var/www/html/pinga to \$BACKUP_DIR."
                        else
                            echo "[INFO] No /var/www/html/pinga directory found, skipping backup."
                        fi

                        echo "[INFO] Deployment preparation completed successfully."
                    '
                """
            } catch (Exception e) {
                echo "[ERROR] Deployment preparation failed."
                echo "[DEBUG] Error: ${e.getMessage()}"
                error "Stage failed: Prepare Deployment."
            }
        }
    }
}

        stage('Deploy New Build') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            echo '[INFO] Removing old deployment...';
                            sudo rm -rf /var/www/html/pinga/
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
    script {
        echo "[ERROR] Deployment failed. Initiating rollback and sending failure notification..."
        // Rollback logic
        sshagent(credentials: [CREDENTIALS_ID]) {
            sh """
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} << 'EOF'
                    # Stop apache service to rollback
                    echo "[INFO] Stopping Apache service..."
                    sudo service apache2 stop || { echo '[ERROR] Apache2 stop failed'; exit 1; }

                    # Locate the most recent backup
                    BACKUP_DIR=\$(ls -td /home/ubuntu/backup-dist-* | head -n 1)
                    if [ -d "\$BACKUP_DIR" ]; then
                        echo "[INFO] Most recent backup found: \$BACKUP_DIR"

                        # Remove the current dist directory
                        echo "[INFO] Removing old dist directory..."
                        sudo rm -rf /var/www/html/ || { echo '[ERROR] Removal failed'; exit 1; }

                        # Restore the most recent backup
                        echo "[INFO] Restoring backup from \$BACKUP_DIR..."
                        sudo cp -r "\$BACKUP_DIR" /var/www/html/ || { echo '[ERROR] Restore failed'; exit 1; }

                        # Update permissions
                        echo "[INFO] Updating permissions..."
                        sudo chown -R www-data:www-data /var/www || { echo '[ERROR] Permissions update failed'; exit 1; }
                        
                        else
                        echo "[ERROR] No backup directory found for rollback."
                        exit 1
                    fi
                    
                    # Restart apache service
                    echo "[INFO] Restarting Apache service..."
                    sudo service apache2 start || { echo '[ERROR] Apache2 start failed'; exit 1; }

                    # Restart apache service
                    echo "[INFO] Restarting Apache service..."
                    sudo systemctl restart apache2 || { echo '[ERROR] Apache2 start failed'; exit 1; }
                EOF
            """
        }
    }
    }
}
}
