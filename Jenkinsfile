// [변수 선언]
def qualityGateResult

pipeline {
    agent {
        docker {
            image 'python:3.10'
            args '-u root:root --network=shared-net'
        }
    }

    environment {
        SWV_BACKEND_URL='http://mp-backend:3000/api'
        PYEXAMINE_URL='http://pyexamine-service:8000/analyze'
        SONAR_PROJECT_KEY='stable-baselines3'
        // [신규 추가] Parser 컨테이너 엔드포인트
        PARSER_URL='http://mp-parser:3001/analyze'

        SONAR_SERVER='SonarQube-Server'
        SONAR_CREDENTIALS='SONAR_QUBE_TOKEN'
        SWV_CREDENTIALS='SWV_BACKEND_TOKEN_ID'
        PYTHONIOENCODING='utf-8'
        
    }
    
    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner-Latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // [신규 추가] Tree-sitter 기반 4계층 구조 분석 및 DB 저장 스테이지
        stage('Environment Setup & Analysis') { // 순서를 맨 앞으로 조정
            steps {
                script {
                    // 1. 도구 설치 확인 (이미지에 zip/curl이 없을 경우 대비)
                    sh 'apt-get update && apt-get install -y zip curl'

                    // 2. 파일 압축
                    echo ">>> Zipping Source Code..."
                    sh 'zip -r code_to_analyze.zip . -x "*.git*" "node_modules/*" "dist/*"'

                    // 3. 파일 생성 여부 즉시 확인 (검증 단계)
                    if (!fileExists('code_to_analyze.zip')) {
                        error "파일 생성 실패: code_to_analyze.zip이 존재하지 않습니다."
                    }

                    // 4. Parser로 전송
                    echo ">>> Sending to Parser Container..."
                    try {
                        // -v 옵션을 추가하여 상세한 통신 로그 확인
                        def parserResponse = sh(
                            script: """
                                curl -v -s -X POST "http://mp-parser:3001/analyze" \
                                -F "file=@code_to_analyze.zip"
                            """, 
                            returnStdout: true
                        ).trim()
                        
                        writeFile file: 'ast_result.json', text: parserResponse
                        echo ">>> AST Data saved to ast_result.json (File size: ${parserResponse.length()} bytes)"
                    } catch (Exception e) {
                        echo ">>> [Critical] Parser Communication Error: ${e.message}"
                        // 파일이 있는데도 에러가 난다면 네트워크 문제임
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    sh 'apt-get update && apt-get install -y default-jre zip curl'
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
                    def scannerHome=tool 'SonarScanner-Latest'
                    
                    withSonarQubeEnv(env.SONAR_SERVER) {
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

                    def pyExamineResponse = sh(script: """
                        curl -X POST "${env.PYEXAMINE_URL}" \
                        -F "file=@source_code.zip" \
                        -H "accept: application/json"
                    """, returnStdout: true).trim()

                    def rawResults = readJSON text: pyExamineResponse
                    
                    def mergedPayload = [
                        teamName: "stable-baselines3", 
                        jenkinsJobName: env.JOB_NAME,
                        sonarProjectKey: env.SONAR_PROJECT_KEY,
                        analysis: [
                            jobName: env.JOB_NAME,
                            buildNumber: env.BUILD_NUMBER.toInteger(),
                            status: "COMPLETED", 
                            buildUrl: env.BUILD_URL,
                            commitHash: sh(returnStdout: true, script: 'git rev-parse HEAD').trim(),
                            pyExamineResult: rawResults
                        ]
                    ]

                    writeJSON file: 'final_payload.json', json: mergedPayload, pretty: 4
                    def payloadString = readFile 'final_payload.json'
                    
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
                        teamName: "stable-baselines3", 
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
                            url: "${env.SWV_BACKEND_URL}/team-projects", 
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