pipeline {
    agent any
   
   

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        BUILD_DATE = "${new Date().format('ddMMMyyyy')}"
        BUILD_DIR = "/home/ubuntu"
        DIST_FILE = ''
        //FRONTEND_SERVER = ''
         FRONTEND_SERVER = 'ssh -i "/home/ubuntu/vkey.pem" ubuntu@ec2-3-110-190-110.ap-south-1.compute.amazonaws.com'  // Set this to your actual frontend server address
        CREDENTIALS_ID = ''
       // NODE_OPTIONS = '--max_old_space_size=4096'  // Increase Node.js memory limit if needed
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
               // case 'prod':
                 //   env.DIST_FILE = "dist-prod-${env.BUILD_DATE}-new.tar.gz"
                  //  env.FRONTEND_SERVER = "crmprod.pingacrm.com"
                  //  env.CREDENTIALS_ID = "prod-frontend-ssh-key"
                  //  break
                default:
                    error "[ERROR] Invalid environment: ${params.ENVIRONMENT}. Use 'dev', 'uat', or 'prod'."
            }
            echo "[DEBUG] ENVIRONMENT=${params.ENVIRONMENT}, DIST_FILE=${env.DIST_FILE}, FRONTEND_SERVER=${env.FRONTEND_SERVER}, CREDENTIALS_ID=${env.CREDENTIALS_ID}"
        }
    }
}


      //  stage('Backup Current Code') {
        //    steps {
        //        echo "[INFO] Backing up current deployment at /home/ubuntu/pinga"
        //        sh "sudo cp -R /home/ubuntu/pinga /home/ubuntu/pinga-${env.BUILD_DATE}-backup || exit 1"
        //        echo "[INFO] Backup completed successfully."
        //    }
      //  }

        stage('Clean Old Build Files') {
            steps {
                echo "[INFO] Cleaning up old build files from previous deployments."
                sh "sudo rm -rf /home/ubuntu/pinga/trunk/dist || exit 1"
                echo "[INFO] Old build files removed."
            }
        }

      //  stage('Fetch Latest Code from SVN') {
     //       steps {
     ////           script {
     ///               echo "[INFO] Fetching latest code from SVN repository."
      ///              def svnUrl = "https://extsvn.pingacrm.com/svn/pingacrm-frontend-new/trunk"

         //           withCredentials([usernamePassword(credentialsId: 'svn-credentials-id', 
        //                                              usernameVariable: 'SVN_USER', 
        //                                              passwordVariable: 'SVN_PASS')]) {
         //               sh """
         //               svn checkout --username ${SVN_USER} --password ${SVN_PASS} ${svnUrl} /home/ubuntu/pinga/trunk || exit 1
        ////                """
        //            }

           //         echo "[INFO] SVN checkout/update completed successfully."
          //      }
        ////    }
   //     }

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
                # Clean up any previous dependencies and lock files
                rm -rf node_modules package-lock.json
                echo "[INFO] Removed existing dependencies."
                
                # Install dependencies with legacy peer dependencies to avoid version conflicts
                npm install --legacy-peer-deps || exit 1
                echo "[INFO] Dependencies installed successfully."
                
                # Fix any vulnerabilities found
                echo "[INFO] Fixing vulnerabilities with npm audit..."
                npm audit fix || echo "Audit fix failed; ignoring remaining issues."
                npm audit fix --force || echo "Force audit fix failed (some breaking changes may remain)."
                
                # Run the build process
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
                // Dynamic variables for tar file name and path
                def ENVIRONMENT = params.ENVIRONMENT
                def DATE = new Date().format("ddMMMyyyy")
                def DIST_FILE = "dist-${ENVIRONMENT}-${DATE}-new.tar.gz"
                def TAR_PATH = "${env.BUILD_DIR}/${DIST_FILE}"

                // Compress artifacts
                sh """
                echo "[INFO] Compressing build artifacts into: ${TAR_PATH}."
                sudo tar -czvf "${TAR_PATH}" dist || exit 1
                echo "[INFO] Build artifacts compressed successfully into ${TAR_PATH}."
                """

                // Upload to S3
                sh """
                echo "[INFO] Uploading ${TAR_PATH} to S3 bucket."
                if ! aws s3 cp "${TAR_PATH}" s3://pinga-builds/; then
                  echo "[ERROR] Failed to upload ${TAR_PATH} to S3. Exiting."
                  exit 1
                else
                  echo "[INFO] Build artifact uploaded successfully to s3://pinga-builds/${DIST_FILE}."
                fi
                """
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
        echo "[INFO] Deploying build artifact to server: ${FRONTEND_SERVER}"
        echo "[DEBUG] ENVIRONMENT=${params.ENVIRONMENT}, CREDENTIALS_ID=${env.CREDENTIALS_ID}"

        // Using credentials to SSH into the frontend server and perform deployment steps
        withCredentials([sshUserPrivateKey(credentialsId: "${env.CREDENTIALS_ID}", keyFileVariable: 'SSH_KEY_PATH')]) {
            sh '''
                # SSH into the server and perform deployment steps dynamically based on the environment
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY_PATH} ubuntu@${FRONTEND_SERVER} << EOF
                echo "[INFO] Stopping Apache server."
                sudo service apache2 stop || exit 1
                echo "[INFO] Apache server stopped."

                echo "[INFO] Downloading build artifact from S3."
                aws s3 cp s3://pinga-builds/${DIST_FILE} . || exit 1
                echo "[INFO] Build artifact downloaded successfully."

                # Backup existing deployment (only for prod environments)
                if [ "${params.ENVIRONMENT}" == "prod" ]; then
                    if [ -d /var/www/html/pinga ]; then
                        echo "[INFO] Backing up existing deployment."
                        sudo mv /var/www/html/pinga /var/www/html/pinga-backup-${BUILD_DATE} || exit 1
                        echo "[INFO] Backup of existing deployment completed."
                    fi
                fi

                # Deploy the new build
                echo "[INFO] Deploying new build to web server."
                mkdir -p /tmp/${ENVIRONMENT}-dist || exit 1
                tar -xvf ${DIST_FILE} -C /tmp/${ENVIRONMENT}-dist || exit 1
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
        withCredentials([sshUserPrivateKey(credentialsId: "${env.CREDENTIALS_ID}", keyFileVariable: 'SSH_KEY_PATH')]) {
            sh '''
                ssh -i ${SSH_KEY_PATH} ubuntu@${FRONTEND_SERVER} << EOF
                # Stop Apache server before rollback
                echo "[INFO] Stopping Apache server for rollback."
                sudo service apache2 stop || { echo "[ERROR] Failed to stop Apache server."; exit 1; }
                
                # Check for backup directory existence
                if [ ! -d /var/www/html/pinga-backup-${BUILD_DATE} ]; then
                    echo "[ERROR] Backup not found. Rollback aborted. Manual intervention required."
                    exit 1
                fi

                script {
    echo "[DEBUG] CREDENTIALS_ID=${env.CREDENTIALS_ID}"
}


                # Perform rollback
                echo "[INFO] Rolling back to previous deployment."
                sudo rm -rf /var/www/html/pinga || { echo "[ERROR] Failed to remove current deployment."; exit 1; }
                sudo mv /var/www/html/pinga-backup-${BUILD_DATE} /var/www/html/pinga || { echo "[ERROR] Failed to restore backup."; exit 1; }
                sudo chown -R www-data:www-data /var/www/html/pinga || { echo "[ERROR] Failed to set permissions on restored files."; exit 1; }

                # Restart Apache server
                echo "[INFO] Restarting Apache server after rollback."
                sudo service apache2 start || { echo "[ERROR] Failed to start Apache server."; exit 1; }

                echo "[INFO] Rollback completed successfully."
                EOF
            '''
        }
    }
}
}
