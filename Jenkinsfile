// [변수 선언] 파이프라인 최상단
def qualityGateResult

properties([
    parameters([
        string(name: 'SONAR_PROJECT_KEY', defaultValue: 'your-python-project-key', description: 'SonarQube Project Key'),
        string(name: 'SWV_BACKEND_URL', defaultValue: 'http://mp-backend:3000/api/team-projects', description: 'SWV Backend Notification URL'),
        // [추가] PyExamine 서비스 URL (Docker Compose 서비스명 사용 가능)
        string(name: 'PYEXAMINE_URL', defaultValue: 'http://pyexamine-service:8000/analyze', description: 'PyExamine Service URL')
    ])
])

pipeline {
    agent {
        docker {
            // [수정 1] Python 환경 이미지로 변경
            image 'python:3.10'
            // [참고] PyExamine 및 Backend와 통신하기 위해 shared-net 연결 유지
            args '--network=shared-net'
        }
    }

    environment {
        SONAR_SERVER       = 'SonarQube-Server'
        SONAR_CREDENTIALS  = 'SONAR_QUBE_TOKEN'
        SWV_CREDENTIALS    = 'SWV_BACKEND_TOKEN_ID'
        // [추가] 한글 인코딩 문제 방지
        PYTHONIOENCODING   = 'utf-8'
    }
    
    // [수정 2] Gradle 없이 SonarQube 분석을 수행하기 위해 스캐너 도구 활성화
    // Jenkins 관리 > Global Tool Configuration에 'SonarScanner'가 설정되어 있어야 합니다.
    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner'
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
                    // [수정 3] Gradle 빌드 대신 pip로 의존성 설치
                    // requirements.txt가 존재할 경우에만 설치
                    if (fileExists('requirements.txt')) {
                        sh 'pip install -r requirements.txt'
                    } else {
                        echo "No requirements.txt found. Skipping dependency installation."
                    }
                    
                    // PyExamine 전송을 위한 zip, curl 설치 (slim 이미지일 경우 필요)
                    sh 'apt-get update && apt-get install -y zip curl'
                }
            }
        }

        // [추가] PyExamine 분석 및 결과 전송 스테이지
        stage('PyExamine Analysis') {
            steps {
                script {
                    echo ">>> Starting PyExamine Analysis..."
                    
                    // 1. 소스 코드 압축 (.git 및 불필요 파일 제외)
                    sh 'zip -r source_code.zip . -x "*.git*" "__pycache__/*" "*.pyc"'

                    // 2. PyExamine 컨테이너로 분석 요청 (JSON 결과 수신)
                    // Docker Network 상의 pyexamine-service:8000 호출
                    def pyExamineResponse = sh(script: """
                        curl -X POST "${params.PYEXAMINE_URL}" \
                        -F "file=@source_code.zip" \
                        -H "accept: application/json"
                    """, returnStdout: true).trim()

                    echo ">>> PyExamine Analysis Completed."
                    
                    // 3. 결과를 백엔드로 전송
                    // 백엔드 URL 끝에 '/metrics'를 붙여 엔드포인트 구성 (백엔드 API 명세에 따라 수정 필요)
                    // 예: http://mp-backend:3000/api/team-projects/metrics
                    def backendMetricsUrl = "${params.SWV_BACKEND_URL}/metrics"
                    
                    try {
                        // 결과 파일 저장 (디버깅용)
                        writeFile file: 'pyexamine_result.json', text: pyExamineResponse
                        
                        echo ">>> Sending PyExamine results to Backend: ${backendMetricsUrl}"
                        
                        // 백엔드로 JSON 전송
                        sh """
                            curl -X POST "${backendMetricsUrl}" \
                            -H "Content-Type: application/json" \
                            -d @pyexamine_result.json
                        """
                        echo ">>> PyExamine results sent successfully."
                    } catch (Exception e) {
                        echo ">>> Failed to send PyExamine results: ${e.getMessage()}"
                        // PyExamine 실패가 전체 파이프라인을 멈추지 않도록 예외 처리
                    }
                }
            }
        }

        stage('SonarQube Analysis & Quality Gate') {
            steps {
                script {
                    withSonarQubeEnv(env.SONAR_SERVER) {
                        // [수정 4] gradlew sonar 대신 sonar-scanner 사용
                        // Python 프로젝트 분석 설정 전달
                        sh """
                            sonar-scanner \
                            -Dsonar.projectKey=${params.SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.token=${SONAR_AUTH_TOKEN} \
                            -Dsonar.language=py \
                            -Dsonar.python.version=3.10
                        """
                    }
                    
                    // Quality Gate 대기
                    qualityGateResult = waitForQualityGate abortPipeline: true, credentialsId: env.SONAR_CREDENTIALS
                }
            }
        }

        stage('Notify SWV Backend') {
            steps {
                script {
                    // 기존 로직 유지 (빌드 상태 및 메타데이터 전송)
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
                    
                    echo "========================================================"
                    echo ">>> Preparing to send Build Status to SWV Backend"
                    echo payloadJson
                    echo "========================================================"
                    
                    try {
                        httpRequest(
                            url: params.SWV_BACKEND_URL,
                            httpMode: 'POST',
                            contentType: 'APPLICATION_JSON',
                            requestBody: payloadJson,
                            quiet: false
                        )
                        echo ">>> Build Status notification sent."
                    } catch (Exception e) {
                        echo ">>> Failed to send Build Status: ${e.getMessage()}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Cleaning up workspace..."
            // 임시 파일 삭제
            sh 'rm -f source_code.zip pyexamine_result.json'
            cleanWs()
        }
    }
}