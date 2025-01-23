
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
    }
}
