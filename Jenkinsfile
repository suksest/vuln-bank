pipeline {
    agent any

    stages {
        stage('Secret Scanning') {
            steps {
                sh 'echo "Secret scanning..."'
                sh trufflehog --no-update git file://. --json | tee trufflehog.json
                archiveArtifacts artifacts: 'trufflehog.json', fingerprint: true
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

