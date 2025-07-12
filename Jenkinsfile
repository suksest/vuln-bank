pipeline {
    agent any

    stages {
        stage('Secret Scanning') {
            steps {
                script {
                    sh 'echo "Secret scanning..."'
                    sh '''
                        trufflehog --no-update git file://. --json | tee trufflehog.json
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trufflehog.json', fingerprint: true
                }
                success {
                    sh 'echo "Secret scanning completed successfully"'
                }
                failure {
                    sh 'echo "Secret scanning failed"'
                }
            }
        }
        stage('SCA') {
            steps {
                sh 'echo "SCA..."'
                sh 'syft scan . -o cyclonedx-json > vuln-bank-syft.json'
                sh 'grype sbom:./vuln-bank-syft.json -o cyclonedx-json > vuln-bank-grype.json'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'vuln-bank-syft.json', fingerprint: true
                    archiveArtifacts artifacts: 'vuln-bank-grype.json', fingerprint: true
                }
                success {
                    sh 'echo "SCA completed successfully"'
                }
                failure {
                    sh 'echo "SCA failed"'
                }
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