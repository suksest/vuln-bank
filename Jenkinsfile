def reportTemplate = "empty"
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
                script {
                    def severityCountsJson = sh(
                        script: "jq 'reduce .vulnerabilities[].ratings[].severity as \\$s ({}; .[\\$s] = (.[\\$s] // 0) + 1)' vuln-bank-grype.json",
                        returnStdout: true
                    ).trim()

                    def counts = new groovy.json.JsonSlurper().parseText(severityCountsJson)
                    def lowCount    = counts.low    ?: 0
                    def medCount    = counts.medium ?: 0
                    def highCount   = counts.high   ?: 0

                    reportTemplate = "SCA: Low: ${lowCount}, Medium: ${medCount}, High: ${highCount}"

                    if (highCount > 0) {
                        unstable("Found vulnerabilities with high severity")
                        echo(reportTemplate)
                    } else if (medCount > 0 || lowCount > 0) {
                        echo(reportTemplate)
                    } else {
                        echo("No vulnerabilities found")
                    }
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
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'sonarqube-results-issues.json', fingerprint: true
                }
                success {
                    sh 'echo "SAST completed successfully"'
                    script {
                        def qualityGate = waitForQualityGate()
                        echo "SonarCloud quality gate status: ${qualityGate.status}"

                        def response = ""
                        withSonarQubeEnv(installationName: 'sonar-vulnbank-devsecops', envOnly: true) {
                            response = sh (
                                script: """
                                    curl -s -H "Authorization: Bearer ${SONAR_AUTH_TOKEN}" \
                                        "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${env.PROJECT_NAME}"
                                """,
                                returnStdout: true
                            ).trim()
                        }

                        def json = readJSON text: response

                        echo "=== Detailed Quality Gate Status ==="
                        echo "Overall Status: ${json.projectStatus.status}"
                        echo "Conditions: ${json.projectStatus.conditions}"
                        echo "=== Detailed Quality Gate Status ==="
                    }
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
                    } else if (zapExitCode == 1) {
                        unstable("ZAP scan completed with blocker issue(s)")
                    } else if (zapExitCode == 2) {
                        unstable("ZAP scan found potential issue(s)")
                    } else {
                        error("ZAP scan failed with exit code ${zapExitCode}")
                    }
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
                    description: "Build ${env.BUILD_DISPLAY_NAME} completed with status ${currentBuild.currentResult}.\n\n${reportTemplate}",
                )
            }
        }
    }
}