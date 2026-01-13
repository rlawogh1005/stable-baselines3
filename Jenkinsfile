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
        SONAR_PROJECT_KEY="${env.JOB_NAME}"
        
        // [주의] 백엔드 API 경로 확인 필요 (로그상 404 발생함)
        SWV_BACKEND_URL='http://mp-backend:3000/api'
        PYEXAMINE_URL='http://pyexamine-service:8000/analyze'

        SONAR_SERVER='SonarQube-Server'
        SONAR_CREDENTIALS='SONAR_QUBE_TOKEN'
        SWV_CREDENTIALS='SWV_BACKEND_TOKEN_ID'
        PYTHONIOENCODING='utf-8'
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
                    // 1. 패키지 설치 (기존 코드)
                    sh 'apt-get update && apt-get install -y default-jre zip curl'
                    
                    // [핵심 추가] Git Dubious Ownership 에러 방지 설정
                    sh 'git config --global --add safe.directory "*"'

                    if (fileExists('requirements.txt')) {
                        sh 'pip install -r requirements.txt'
                    }
                }
            }
        }

        stage('SonarQube Analysis & Quality Gate') {
            steps {
                script {
                    // [핵심 수정] 도구 경로를 명시적으로 변수에 할당
                    def scannerHome=tool 'SonarScanner-Latest'
                    
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
                    qualityGateResult=waitForQualityGate abortPipeline: true, credentialsId: env.SONAR_CREDENTIALS
                }
            }   
        }

        stage('PyExamine Analysis') {
            steps {
                script {
                    echo ">>> Starting PyExamine Analysis..."
                    sh 'zip -r source_code.zip . -x "*.git*" "__pycache__/*" "*.pyc"'

                    // 1. PyExamine Raw 데이터 수신
                    def pyExamineResponse = sh(script: """
                        curl -X POST "${env.PYEXAMINE_URL}" \
                        -F "file=@source_code.zip" \
                        -H "accept: application/json"
                    """, returnStdout: true).trim()

                    // 2. 데이터 파싱 (readJSON 사용 - 안전함)
                    def rawResults = readJSON text: pyExamineResponse
                    
                    // 3. Payload 구성
                    def mergedPayload = [
                        teamName: "stable-baselines3", 
                        jenkinsJobName: env.JOB_NAME,
                        analysis: [
                            jobName: env.JOB_NAME,
                            buildNumber: env.BUILD_NUMBER.toInteger(),
                            status: "COMPLETED", 
                            buildUrl: env.BUILD_URL,
                            commitHash: sh(returnStdout: true, script: 'git rev-parse HEAD').trim(),
                            pyExamineResult: rawResults
                        ]
                    ]

                    // [핵심 수정] 4. 파일 저장 및 JSON 생성 (writeJSON 사용)
                    // groovy.json.JsonOutput 대신 writeJSON 단계를 사용하여 StackOverflow 방지
                    writeJSON file: 'final_payload.json', json: mergedPayload, pretty: 4
                    
                    // 5. 로깅을 위해 파일 내용을 다시 읽음 (직렬화 부하 없음)
                    def payloadString = readFile 'final_payload.json'
                    
                    echo "========================================================"
                    echo "   [DEBUG] FINAL PAYLOAD TO BACKEND"
                    echo "========================================================"
                    echo "1. Target Endpoint: ${env.SWV_BACKEND_URL}/team-projects"
                    echo "2. Payload Preview:"
                    echo payloadString
                    echo "========================================================"

                    // 6. 백엔드로 전송
                    def backendUrl = "${env.SWV_BACKEND_URL}/team-projects"
                    
                    sh """
                        curl -X POST "${backendUrl}" \
                        -H "Content-Type: application/json" \
                        -d @final_payload.json
                    """
                }
            }
        }

        stage('Notify SWV Backend') {
            steps {
                script {
                    def payload=[
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
                    def payloadJson=groovy.json.JsonOutput.toJson(payload)
                    
                    try {
                        httpRequest(
                            url: env.SWV_BACKEND_URL/'static-analysis', 
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