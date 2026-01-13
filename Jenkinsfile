// [변수 선언]
def qualityGateResult

pipeline {
    agent {
        docker {
            image 'python:3.10'
            // [유지] 루트 권한 및 네트워크 설정
            args '-u root:root --network=shared-net'
        }
    }

    environment {
        // [자동 설정] SonarQube 키를 Jenkins Job 이름으로 설정
        SONAR_PROJECT_KEY  = "${env.JOB_NAME}"
        
        // [주의] 백엔드 API 경로 확인 필요 (로그상 404 발생함)
        SWV_BACKEND_URL    = 'http://mp-backend:3000/api/team-projects'
        PYEXAMINE_URL      = 'http://pyexamine-service:8000/analyze'

        SONAR_SERVER       = 'SonarQube-Server'
        SONAR_CREDENTIALS  = 'SONAR_QUBE_TOKEN'
        SWV_CREDENTIALS    = 'SWV_BACKEND_TOKEN_ID'
        PYTHONIOENCODING   = 'utf-8'
    }
    
    // [참고] tools 블록은 유지하되, 실제 실행은 경로를 받아와서 수행함
    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner-Latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    // [유지] Java 필수 설치
                    sh 'apt-get update && apt-get install -y default-jre zip curl'
                    
                    if (fileExists('requirements.txt')) {
                        sh 'pip install -r requirements.txt'
                    }
                }
            }
        }

        stage('PyExamine Analysis') {
            steps {
                script {
                    echo ">>> Starting PyExamine Analysis..."
                    sh 'zip -r source_code.zip . -x "*.git*" "__pycache__/*" "*.pyc"'

                    def pyExamineResponse = sh(script: """
                        curl -X POST "${env.PYEXAMINE_URL}" \
                        -F "file=@source_code.zip" \
                        -H "accept: application/json"
                    """, returnStdout: true).trim()

                    echo ">>> PyExamine Analysis Completed."
                    
                    def backendMetricsUrl = "${env.SWV_BACKEND_URL}/metrics"
                    
                    try {
                        writeFile file: 'pyexamine_result.json', text: pyExamineResponse
                        // [알림] 이전 빌드에서 여기서 404가 떴습니다. 백엔드 URL을 확인하세요.
                        sh """
                            curl -X POST "${backendMetricsUrl}" \
                            -H "Content-Type: application/json" \
                            -d @pyexamine_result.json
                        """
                        echo ">>> PyExamine results sent successfully."
                    } catch (Exception e) {
                        echo ">>> Failed to send: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('SonarQube Analysis & Quality Gate') {
            steps {
                script {
                    // [핵심 수정] 도구 경로를 명시적으로 변수에 할당
                    def scannerHome = tool 'SonarScanner-Latest'
                    
                    withSonarQubeEnv(env.SONAR_SERVER) {
                        // [핵심 수정] 절대 경로로 실행 (${scannerHome}/bin/sonar-scanner)
                        sh """
                            "${scannerHome}/bin/sonar-scanner" \
                            -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.token=${SONAR_AUTH_TOKEN} \
                            -Dsonar.language=py \
                            -Dsonar.python.version=3.10
                        """
                    }
                    qualityGateResult = waitForQualityGate abortPipeline: true, credentialsId: env.SONAR_CREDENTIALS
                }
            }
        }

        stage('Notify SWV Backend') {
            steps {
                script {
                    def payload = [
                        teamName: "IEUM-Backend-Team", 
                        jenkinsJobName: env.JOB_NAME,
                        analysis: [
                            jobName: env.JOB_NAME,
                            buildNumber: env.BUILD_NUMBER.toInteger(),
                            status: qualityGateResult.status,
                            buildUrl: env.BUILD_URL,
                            commitHash: sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                        ]
                    ]
                    def payloadJson = groovy.json.JsonOutput.toJson(payload)
                    
                    try {
                        httpRequest(
                            url: env.SWV_BACKEND_URL, 
                            httpMode: 'POST',
                            contentType: 'APPLICATION_JSON',
                            requestBody: payloadJson,
                            quiet: false
                        )
                    } catch (Exception e) {
                        echo ">>> Failed to send Build Status."
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f source_code.zip pyexamine_result.json'
            cleanWs()
        }
    }
}