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
        SONAR_PROJECT_KEY="${env.JOB_NAME}"
        SWV_BACKEND_URL='http://mp-backend:3000/api'
        PYEXAMINE_URL='http://pyexamine-service:8000/analyze'
        
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
        stage('Code Structure Analysis (Tree-sitter)') {
            steps {
                script {
                    echo ">>> Triggering Tree-sitter Parser for Structure Analysis..."
                    
                    // Parser 컨테이너에게 현재 마운트된 소스 경로(/code) 분석 요청
                    // Parser는 분석 후 스스로 mp-backend에 결과를 전송함
                    def parserResponse = sh(script: """
                        curl -X POST "${env.PARSER_URL}" \
                        -H "Content-Type: application/json" \
                        -d '{"path": "/code"}'
                    """, returnStdout: true).trim()
                    
                    echo "Parser Response: ${parserResponse}"
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