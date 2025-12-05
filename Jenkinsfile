pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
                sh 'echo "Compilando proyecto..."'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                // Este comando fallará y generará un error
                sh '''
                    echo "Ejecutando tests..."
                    echo "ERROR: Tests failed: 3/10 tests passed"
                    exit 1
                '''
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'This stage will not run due to test failure'
            }
        }
    }
    
    post {
        failure {
            echo 'Pipeline failed! Sending logs to monitoring system...'
        }
    }
}
