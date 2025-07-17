def reportTemplate = "Security Scan Report"
pipeline {
    agent any
    environment {
        PROJECT_NAME = "vuln-bank"
    }
    
    stages {
        stage('Secret Scanning') {
            steps {
                script {
                    sh 'echo "Secret scanning..."'
                    sh '''
                        trufflehog --no-update git file://. --json | tee trufflehog-git.json
                    '''

                    script {
                        def secretsCountGit = 0
                        if (fileExists('trufflehog-git.json')) {
                            secretsCountGit = sh(script: "grep -c '^{' trufflehog-git.json || true", returnStdout: true).trim()
                        }

                        def secretsReport = "🔑 **Secret Scan**: ${secretsCountGit} secrets found"
                        reportTemplate = secretsReport
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trufflehog-git.json', fingerprint: true
                    archiveArtifacts artifacts: 'trufflehog-filesystem.json', fingerprint: true
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
                script {
                    def totalVulns = sh(
                        script: """
                            jq '.vulnerabilities | length' vuln-bank-grype.json
                        """,
                        returnStdout: true
                    ).trim()
                    
                    def scaReport = "📦 **SCA**: ${totalVulns} vulnerabilities"
                    reportTemplate = reportTemplate + "\n" + scaReport
                }
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
                        -Dsonar.projectKey=${env.PROJECT_NAME} \
                        -Dsonar.organization=suksest \
                        -Dsonar.sources=. \
                        -Dsonar.excludes=*.json \
                        -Dsonar.python.version=3.9"""
                    }

                    withSonarQubeEnv(installationName: 'sonar-vulnbank-devsecops', envOnly: true) {
                        sh """
                            curl -s -H "Authorization: Bearer ${SONAR_AUTH_TOKEN}" \
                                "${SONAR_HOST_URL}/api/issues/search?projectKeys=${env.PROJECT_NAME}" \
                                > sonarqube-results-issues.json || true
                        """
                    }

                    def sastCount = 0
                    if (fileExists('sonarqube-results-issues.json')) {
                        sastCount = sh(script: "jq '.total' sonarqube-results-issues.json", returnStdout: true).trim()
                    }

                    def sastReport = "🔍 **SAST**: ${sastCount} issues found"
                    reportTemplate = reportTemplate + "\n" + sastReport
                }
            }
            post {
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
        stage('Build & Deploy to Staging') {
            steps {
                script {
                    sh 'echo "Starting application container..."'
                    sh """
                        docker compose -f docker-compose-ci.yml up -d --build --remove-orphans

                        docker image prune -f
                    """
                    
                    sh "sleep 30"
                    sh "docker compose -f docker-compose-ci.yml logs"
                }
            }
        }
        stage('DAST') {
            steps {
                script {
                    def APP_IP = sh(
                        script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${env.PROJECT_NAME}-web", 
                        returnStdout: true
                    ).trim()
                    
                    def TARGET_URL = "http://${APP_IP}:5000"
                    def dastReport = "empty"
                    
                    sh "echo 'Target application URL: ${TARGET_URL}'"

                    def zapExitCode = sh(
                        script: """
                            docker run --name zap-${env.BUILD_ID} -u root -p 8090:8080 \
                            -v ${WORKSPACE}:/zap/wrk/ --network zapnet \
                            -i zaproxy/zap-stable \
                            zap-baseline.py -t ${TARGET_URL} -J zap-report.json
                        """,
                        returnStatus: true
                    )

                    sh "docker cp zap-${env.BUILD_ID}:/zap/wrk/zap-report.json ."

                    if (zapExitCode == 0) {
                        echo "ZAP scan completed successfully with no warnings"
                        dastReport = "🚨 **DAST**: PASSED ✅"
                    } else if (zapExitCode == 1) {
                        unstable("ZAP scan completed with blocker issue(s)")
                        dastReport = "🚨 **DAST**: NOT PASSED ❌"
                    } else if (zapExitCode == 2) {
                        unstable("ZAP scan found potential issue(s)")
                        dastReport = "🚨 **DAST**: NOT PASSED ❌"
                    } else {
                        error("ZAP scan failed with exit code ${zapExitCode}")
                    }

                    reportTemplate = reportTemplate + "\n" + dastReport
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.json', fingerprint: true
                    
                    // Cleanup containers
                    sh "docker stop zap-${env.BUILD_ID} || true"
                    sh "docker rm zap-${env.BUILD_ID} || true"
                    sh "docker compose -f docker-compose-ci.yml down || true"
                }
            }
        }
    }

    post {
        always {
            withCredentials([string(credentialsId: 'discord-webhook', variable: 'DISCORD_WEBHOOK')]) {
                discordSend(
                    title: "Build ${env.BUILD_DISPLAY_NAME} completed with status ${currentBuild.currentResult}.",
                    webhookURL: DISCORD_WEBHOOK,
                    description: "Build ${env.BUILD_DISPLAY_NAME} completed with status ${currentBuild.currentResult}.\n\n${reportTemplate}\n\n${env.BUILD_URL}",
                    unstable: true,
                    result: currentBuild.currentResult
                )
            }
        }
        success {
            cleanWs()
        }
    }
}