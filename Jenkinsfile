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
        stage('SAST') {
            steps {
                sh 'echo "SAST..."'
                script {
                    def scannerHome = tool 'sonarscanner'
                    withSonarQubeEnv('sonar-vulnbank-devsecops') {
                        sh """${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=${params.PROJECT_NAME} \
                        -Dsonar.organization=suksest \
                        -Dsonar.sources=. \
                        -Dsonar.python.version=3.9"""
                    }

                    withCredentials([string(credentialsId: 'sonar-vulnbank-devsecops', variable: 'SONAR_TOKEN')]) {
                        sh """
                            curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \
                                "${SONAR_HOST_URL}/api/issues/search?projectKeys=${params.PROJECT_NAME}" \
                                > sonarqube-results/issues.json || true
                        """
                    }
                }
                script {
                    sh """
                        curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \
                            "${SONAR_HOST_URL}/api/issues/search?projectKeys=${SONAR_PROJECT_KEY}" \
                            > sonarqube-results-issues.json || true
                    """
                }
            }
            post {sonarqube-results/
                always {
                    archiveArtifacts artifacts: 'sonarqube-results-issues.json', fingerprint: true
                }
                success {
                    sh 'echo "SAST completed successfully"'
                }
                failure {
                    sh 'echo "SAST failed"'
                }
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