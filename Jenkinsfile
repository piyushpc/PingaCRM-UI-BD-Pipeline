pipeline {
    agent { label 'build-server' }
    stages {
        stage('Test Build Server') {
            steps {
                sh '''
                    echo "Agent is successfully connected!"
                    echo "Running on $(hostname)"
                '''
            }
        }
    }
}
