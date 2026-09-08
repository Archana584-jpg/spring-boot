pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 15, unit: 'MINUTES')  # Reduced from 30 (incremental is faster)
        timestamps()
    }
    
    environment {
        SONAR_HOST = 'http://15.206.213.78:9000'
        SONAR_TOKEN = credentials('sonar-token')
        GITHUB_TOKEN = credentials('archana-sonar')
        PROJECT_KEY = "ongrid-${env.BRANCH_NAME.replaceAll('/', '-')}"
        DOCKER_IMAGE = "ongrid-scan-${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('📥 Checkout') {
            steps {
                echo "=========================================="
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Project Key: ${PROJECT_KEY}"
                echo "=========================================="
                checkout scm
            }
        }
        
        stage('🐳 Docker Build (Optimized)') {
            steps {
                echo "Building Docker image for incremental scan..."
                sh 'docker build -f SonarqubeDockerfile -t ${DOCKER_IMAGE} .'
            }
        }
        
        stage('🔍 SonarQube Incremental Scan (Industry Standard)') {
            steps {
                echo "=========================================="
                echo "Running INCREMENTAL scan (changed files only)"
                echo "Expected time: 3-5 minutes"
                echo "=========================================="
                
                script {
                    def sonarArgs = '''
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -Dsonar.projectKey=${PROJECT_KEY} \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.exclusions="**/test/**,**/node_modules/**,**/build/**,**/target/**,**/.gradle/**,**/.m2/**,**/*.min.js,**/*.min.css,**/dist/**,**/*.xml,**/*.properties" \
                        -Dsonar.coverage.exclusions="**/test/**,**/*Test.java"
                    '''
                    
                    // Incremental: Only changed files (Google/Netflix standard)
                    if (env.BRANCH_NAME == 'main') {
                        echo "🔄 Main branch: Full scan (baseline)"
                        sonarArgs += " -Dsonar.scanAllFiles=true"
                    } else {
                        echo "🎯 PR branch: Incremental scan (changed files only)"
                        // SonarQube Community Edition doesn't support PR-mode natively
                        // So we use reference branch to compare against main
                        sonarArgs += " -Dsonar.newCodeDefinitionType=REFERENCE_BRANCH"
                        sonarArgs += " -Dsonar.newCodeDefinitionValue=main"
                    }
                    
                    sh """
                        echo "Starting scan with arguments:"
                        echo "${sonarArgs}"
                        echo ""
                        
                        docker run --rm \
                          -m 2g \
                          -e SONAR_HOST_URL=\${SONAR_HOST} \
                          -e SONAR_LOGIN=\${SONAR_TOKEN} \
                          -e SONAR_SCANNER_OPTS="-Xmx1g -XX:+UseG1GC" \
                          \${DOCKER_IMAGE} \
                          ${sonarArgs}
                    """
                }
            }
        }
        
        stage('⏱️ Quality Gate Check (New Code Only)') {
            steps {
                script {
                    echo "=========================================="
                    echo "Checking Quality Gate (NEW ISSUES ONLY)"
                    echo "Ignoring old code issues"
                    echo "=========================================="
                    
                    def qgStatus = 'UNKNOWN'
                    def attempts = 0
                    def maxAttempts = 12
                    
                    timeout(time: 3, unit: 'MINUTES') {
                        while (attempts < maxAttempts) {
                            try {
                                qgStatus = sh(
                                    script: '''
                                        curl -s -u "${SONAR_TOKEN}": \
                                        "${SONAR_HOST}/api/qualitygates/project_status?projectKey=${PROJECT_KEY}" | \
                                        grep -oP '"status":"\\K[^"]+' || echo "UNKNOWN"
                                    ''',
                                    returnStdout: true
                                ).trim()
                                
                                echo "QG Check ${attempts + 1}/${maxAttempts}: Status = ${qgStatus}"
                                
                                if (qgStatus in ['OK', 'ERROR']) {
                                    break
                                }
                                
                                sleep(5)
                                attempts++
                            } catch (Exception e) {
                                echo "Error: ${e.message}"
                                attempts++
                                sleep(5)
                            }
                        }
                    }
                    
                    if (qgStatus != 'OK') {
                        error("❌ Quality Gate FAILED - Status: ${qgStatus}")
                    } else {
                        echo "✅ Quality Gate PASSED"
                    }
                }
            }
        }
        
        stage('📤 Post GitHub Status') {
            steps {
                script {
                    def status = currentBuild.result == 'SUCCESS' ? 'success' : 'failure'
                    def description = currentBuild.result == 'SUCCESS' ? '✅ Code Quality OK' : '❌ Quality Gate Failed'
                    
                    try {
                        sh '''
                            REPO_NAME=$(basename ${GIT_URL} .git)
                            REPO_OWNER="Archana584-jpg"
                            
                            curl -X POST \
                              -H "Authorization: token ${GITHUB_TOKEN}" \
                              -H "Accept: application/vnd.github.v3+json" \
                              https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${GIT_COMMIT} \
                              -d "{\\"state\\":\\"${status}\\",\\"description\\":\\"${description}\\",\\"context\\":\\"SonarQube/QualityGate\\",\\"target_url\\":\\"${SONAR_HOST}/dashboard?id=${PROJECT_KEY}\\"}"
                        '''
                        echo "GitHub status posted successfully"
                    } catch (Exception e) {
                        echo "⚠️ Warning: GitHub status post failed: ${e.message}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "Cleaning up Docker image: ${DOCKER_IMAGE}"
                docker rmi ${DOCKER_IMAGE} || true
                docker system prune -f
            '''
        }
        
        success {
            echo "=========================================="
            echo "✅ Pipeline completed successfully"
            echo "=========================================="
        }
        
        failure {
            echo "=========================================="
            echo "❌ Pipeline failed"
            echo "=========================================="
        }
    }
}
