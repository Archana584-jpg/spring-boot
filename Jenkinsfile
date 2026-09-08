pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 15, unit: 'MINUTES')
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
                echo "Building Docker image..."
                sh 'docker build -f SonarqubeDockerfile -t ${DOCKER_IMAGE} .'
            }
        }
        
        stage('🔍 SonarQube Incremental Scan') {
            steps {
                echo "=========================================="
                echo "Running INCREMENTAL source-only scan"
                echo "Expected time: 3-5 minutes"
                echo "=========================================="
                
                sh '''
                    docker run --rm \
                      -m 2g \
                      -e SONAR_HOST_URL=${SONAR_HOST} \
                      -e SONAR_LOGIN=${SONAR_TOKEN} \
                      -e SONAR_SCANNER_OPTS="-Xmx1g -XX:+UseG1GC" \
                      ${DOCKER_IMAGE} \
                      -Dsonar.host.url=${SONAR_HOST} \
                      -Dsonar.login=${SONAR_TOKEN} \
                      -Dsonar.projectKey=${PROJECT_KEY} \
                      -Dsonar.sourceEncoding=UTF-8 \
                      -Dsonar.sources=src \
                      -Dsonar.exclusions="**/*.java,**/test/**,**/node_modules/**,**/build/**,**/target/**,**/.gradle/**,**/.m2/**,**/*.min.js,**/*.min.css,**/dist/**" \
                      -Dsonar.java.binaries=.
                '''
            }
        }
        
        stage('⏱️ Quality Gate Check') {
            steps {
                script {
                    echo "=========================================="
                    echo "Checking Quality Gate"
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
                        error("Quality Gate FAILED - Status: ${qgStatus}")
                    } else {
                        echo "Quality Gate PASSED"
                    }
                }
            }
        }
        
        stage('📤 Post GitHub Status') {
            steps {
                script {
                    def status = currentBuild.result == 'SUCCESS' ? 'success' : 'failure'
                    def description = currentBuild.result == 'SUCCESS' ? 'Code Quality OK' : 'Quality Gate Failed'
                    
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
                        echo "GitHub status posted"
                    } catch (Exception e) {
                        echo "Warning: GitHub status post failed"
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                docker rmi ${DOCKER_IMAGE} || true
                docker system prune -f
            '''
        }
        
        success {
            echo "Pipeline completed successfully"
        }
        
        failure {
            echo "Pipeline failed"
        }
    }
}
