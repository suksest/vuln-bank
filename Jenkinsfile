pipeline {
    agent any
    parameters {
        string(name: 'PROJECT_NAME', defaultValue: 'vuln-bank', description: 'Dependency-Track project name')
        string(name: 'PROJECT_VERSION', defaultValue: '1.0.0', description: 'Dependency-Track project version')
    }
    environment {
        DEPENDENCY_TRACK_URL = 'http://192.168.56.16:8082'
        DEPENDENCY_TRACK_API_KEY = credentials('dependency-track-api-key')
    }
    
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
        stage('Upload to Dependency-Track') {
            steps {
                script {
                    sh 'echo "Uploading Grype results to Dependency-Track..."'
                    sh """
                        curl -X POST "${DEPENDENCY_TRACK_URL}/api/v1/bom" \\
                             -H "Content-Type: multipart/form-data" \\
                             -H "X-Api-Key: ${DEPENDENCY_TRACK_API_KEY}" \\
                             -F "projectName=${params.PROJECT_NAME}" \\
                             -F "projectVersion=${params.PROJECT_VERSION}" \\
                             -F "bom=@./vuln-bank-grype.json"
                    """
                }
            }
            post {
                success {
                    sh 'echo "Successfully uploaded to Dependency-Track"'
                }
                failure {
                    sh 'echo "Dependency-Track upload failed"'
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