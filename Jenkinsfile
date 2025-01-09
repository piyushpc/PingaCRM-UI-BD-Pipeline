pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = sh(script: "date +'%d%b%Y'", returnStdout: true).trim() // Dynamically fetch the current build date
        DIST_FILE = "dist-${params.ENVIRONMENT}-${sh(script: 'date +\"%d%b%Y\"', returnStdout: true).trim()}-new.tar.gz"
        BUILD_DIR = "/home/ubuntu"
        //DIST_FILE = '' // Placeholder, will be set dynamically
      //  DIST_FILE = "env.DIST_FILE"
        FRONTEND_SERVER = 'ec2-13-126-252-141.ap-south-1.compute.amazonaws.com'
        CREDENTIALS_ID = 'ubuntu'
        S3_BUCKET = 'pinga-builds'
        SSH_KEY_PATH = '/home/ubuntu/vkey.pem'
        SLACK_CHANNEL = "jenkins"
        DEPLOY_ENV = "${params.ENVIRONMENT}"
        SHELL = '/bin/bash'
        SSH_SESSION_NAME = "jenkins-deployment-${BUILD_ID}"
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
                            env.FRONTEND_SERVER = "ec2-13-126-252-141.ap-south-1.compute.amazonaws.com"
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

        stage('Setup Build Variables') {
            steps {
                script {
                    // Generate the build date in the desired format
                    def BUILD_DATE = sh(script: "date +'%d%b%Y'", returnStdout: true).trim()
                    
                    // Dynamically set DIST_FILE
                    def DIST_FILE = "dist-${params.ENVIRONMENT}-${BUILD_DATE}-new.tar.gz"
                    
                    // Logging
                    echo "[INFO] Selected Environment: ${params.ENVIRONMENT}"
                    echo "[INFO] Build Date: ${BUILD_DATE}"
                    echo "[INFO] Dist File: ${DIST_FILE}"

                    // Set DIST_FILE as an environment variable to be used in later stages
                    env.DIST_FILE = DIST_FILE
                }
            }
        }
        
        stage('Compress & Upload Build Artifacts') {
            steps {
                dir("${env.BUILD_DIR}/pinga/trunk") {
                    echo "[INFO] Compressing and uploading build artifacts..."
                    script {
                        // Use DIST_FILE from the previous step
                        def TAR_PATH = "${env.BUILD_DIR}/${env.DIST_FILE}"
                        
                        // Compress the build artifacts
                        sh "sudo tar -czvf ${TAR_PATH} dist || exit 1"
                        
                        // Upload to S3
                        sh """
                            aws s3 cp ${TAR_PATH} s3://${env.S3_BUCKET}/${env.DIST_FILE} || \
                            { echo '[ERROR] S3 upload failed'; exit 1; }
                        """
                        echo "[INFO] Build artifact uploaded to S3 as: ${env.DIST_FILE}"
                    }
                }
            }
        }

        stage('Start Persistent SSH Session') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        echo "[INFO] Starting persistent SSH session..."
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            tmux new-session -d -s ${SSH_SESSION_NAME} || { echo '[ERROR] Failed to start tmux session'; exit 1; }
                        "
                        echo "[INFO] Persistent SSH session started."
                    """
                }
            }
        }

        stage('Verify Server Availability') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            tmux send-keys -t ${SSH_SESSION_NAME} 'echo [INFO] Server is reachable.' Enter
                        "
                    """
                }
            }
        }

        stage('Stop Apache') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            tmux send-keys -t ${SSH_SESSION_NAME} 'sudo service apache2 stop || { echo [ERROR] Failed to stop Apache; exit 1; }' Enter
                        "
                    """
                }
            }
        }

        stage('Set DIST_FILE Dynamically') {
            steps {
                script {
                    def s3ListOutput = sh(
                        script: "aws s3 ls s3://${S3_BUCKET}/ --region ${AWS_DEFAULT_REGION}",
                        returnStdout: true
                    ).trim()
                    echo "S3 Bucket Contents:\n${s3ListOutput}"

                    def latestFile = sh(
                        script: """
                            aws s3 ls s3://${S3_BUCKET}/ --region ${AWS_DEFAULT_REGION} | \
                            grep '.tar.gz' | sort | tail -n 1 | awk '{print \$4}'
                        """,
                        returnStdout: true
                    ).trim()

                    if (latestFile) {
                        echo "Latest file in S3 bucket: ${latestFile}"
                        env.DIST_FILE = latestFile
                    } else {
                        error "No matching files found in S3 bucket: ${S3_BUCKET}"
                    }
                }
            }
        }

        stage('Download Build from S3') {
    steps {
        sshagent(credentials: [env.CREDENTIALS_ID]) {
            script {
                // Ensure that S3_BUCKET and DIST_FILE are being resolved correctly
                echo "[INFO] Attempting to download from: s3://${env.S3_BUCKET}/${env.DIST_FILE}"
                
                // Check if the file exists in the S3 bucket before downloading
                def checkFileExists = sh(script: "aws s3 ls s3://${env.S3_BUCKET}/${env.DIST_FILE}", returnStatus: true)
                
                if (checkFileExists != 0) {
                    echo "[ERROR] File does not exist in S3: s3://${env.S3_BUCKET}/${env.DIST_FILE}"
                    error "[ERROR] Aborting due to missing file in S3."
                }
                
                // Download the file from S3
                if (sh(script: "aws s3 cp s3://${env.S3_BUCKET}/${env.DIST_FILE} .", returnStatus: true) != 0) {
                    echo "[ERROR] S3 download failed but continuing..."
                    // Fail the build if download fails
                    error "[ERROR] Failed to download ${env.DIST_FILE} from S3"
                }
                
                echo "[INFO] Successfully downloaded: ${env.DIST_FILE}"
            }
        }
  


        stage('Prepare Deployment') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            tmux has-session -t ${SSH_SESSION_NAME} 2>/dev/null || tmux new-session -d -s ${SSH_SESSION_NAME};
                            tmux send-keys -t ${SSH_SESSION_NAME} 'mkdir -p /tmp/${params.ENVIRONMENT}-dist && tar -xvf ${env.DIST_FILE} -C /tmp/${params.ENVIRONMENT}-dist || { echo [ERROR] Unzipping failed; exit 1; }' Enter
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
                            tmux has-session -t ${SSH_SESSION_NAME} 2>/dev/null || tmux new-session -d -s ${SSH_SESSION_NAME};
                            tmux send-keys -t ${SSH_SESSION_NAME} 'sudo service apache2 start || { echo [ERROR] Failed to start Apache; exit 1; }' Enter
                        "
                    """
                }
            }
        }

        stage('Stop Persistent SSH Session') {
            steps {
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                            tmux kill-session -t ${SSH_SESSION_NAME} || echo '[INFO] No tmux session to terminate';
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
                            sudo rm -rf /tmp/${params.ENVIRONMENT}-dist || echo '[INFO] No temporary files to remove';
                        "
                    """
                }
            }
        }

        stage('Smoke Tests') {
            steps {
                script {
                    echo "[INFO] Running smoke tests..."
                    sshagent(credentials: [env.CREDENTIALS_ID]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                        curl -sSf https://crmdev.pingacrm.com | grep -q "<title>Pinga CRM</title>" || { echo '[ERROR] Smoke test failed: Content validation failed'; exit 1; }
                        echo '[INFO] Smoke tests passed successfully.';
                        "
                        """
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
                    body: """
                    Deployment for ${params.ENVIRONMENT} completed successfully at ${new Date()}.
                    Check the application: https://crmdev.pingacrm.com
                    """,
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                slackSend(
                    color: 'good',
                    message: "PingaCRM Deployment Successful for ${params.ENVIRONMENT} :white_check_mark:"
                )
            }
        }
        failure {
            script {
                echo "[ERROR] Deployment failed. Initiating rollback and sending failure notification..."
                sshagent(credentials: [env.CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${env.FRONTEND_SERVER} "
                        sudo service apache2 stop || echo '[INFO] Apache was not running';
                        if [ -d /var/www/html/pinga-backup-${env.BUILD_DATE} ]; then
                            sudo rm -rf /var/www/html/pinga || echo '[INFO] No existing deployment to remove';
                            sudo mv /var/www/html/pinga-backup-${env.BUILD_DATE} /var/www/html/pinga || echo '[ERROR] Rollback failed';
                        fi
                        sudo service apache2 start || echo '[ERROR] Failed to restart Apache';
                        "
                    """
                }
                emailext(
                    subject: "PingaCRM Deployment Failed",
                    body: """
                    Deployment for ${params.ENVIRONMENT} failed. Rollback has been initiated.
                    Please check the Jenkins logs for details.
                    """,
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                slackSend(
                    color: 'danger',
                    message: "PingaCRM Deployment Failed for ${params.ENVIRONMENT}. Rollback initiated. :x:"
                )
            }
        }
    }
