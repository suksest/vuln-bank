pipeline {
    agent any

    stages {
        stage('Secret Scanning') {
            steps {
                sh 'echo "Secret scanning..."'
            }
        }
        stage('SCA') {
            steps {
                sh 'echo "SCA..."'
            }
        }
        stage('SAST') {
            steps {
                sh 'echo "SAST..."'
            }
        }
        stage('DAST') {
            steps {
                sh 'echo "DAST..."'
            }
        }
        stage('Discord Notification') {
            steps {
                sh 'echo "Sending discord notification..."'
            }
        }
    }
}

