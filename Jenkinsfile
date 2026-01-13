// [수정] 변수 선언 제거 (environment에서 처리)
def qualityGateResult

// [핵심 수정 1] parameters 블록 전체 삭제 -> 'Build Now' 클릭 시 즉시 실행됨

pipeline {
    agent {
        docker {
            image 'python:3.10'
            // [권장] 루트 권한으로 실행 (패키지 설치 위해)
            args '-u root:root --network=shared-net'
        }
    }

    environment {
        // [핵심 수정 2] 파라미터 대신 고정된 환경 변수 사용
        // 1. SonarQube 키를 'Jenkins Job 이름'으로 자동 설정
        SONAR_PROJECT_KEY  = "${env.JOB_NAME}"
        
        // 2. URL 설정 고정 (필요시 수정)
        SWV_BACKEND_URL    = 'http://mp-backend:3000/api/team-projects'
        PYEXAMINE_URL      = 'http://pyexamine-service:8000/analyze'

        SONAR_SERVER       = 'SonarQube-Server'
        SONAR_CREDENTIALS  = 'SONAR_QUBE_TOKEN'
        SWV_CREDENTIALS    = 'SWV_BACKEND_TOKEN_ID'
        PYTHONIOENCODING   = 'utf-8'
    }
    
    tools {
        // [유지] 앞서 확인한 도구 이름
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
                    // [유지] Java 오류 해결을 위한 default-jre 사용
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
                    // [유지] 불필요 파일 제외하고 압축
                    sh 'zip -r source_code.zip . -x "*.git*" "__pycache__/*" "*.pyc"'

                    // 환경 변수(PYEXAMINE_URL) 사용
                    def pyExamineResponse = sh(script: """
                        curl -X POST "${env.PYEXAMINE_URL}" \
                        -F "file=@source_code.zip" \
                        -H "accept: application/json"
                    """, returnStdout: true).trim()

                    echo ">>> PyExamine Analysis Completed."
                    
                    def backendMetricsUrl = "${env.SWV_BACKEND_URL}/metrics"
                    
                    try {
                        writeFile file: 'pyexamine_result.json', text: pyExamineResponse
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
                    withSonarQubeEnv(env.SONAR_SERVER) {
                        // [핵심 수정 3] params.KEY 대신 env.SONAR_PROJECT_KEY (즉, JOB_NAME) 사용
                        sh """
                            sonar-scanner \
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
                            url: env.SWV_BACKEND_URL, // 환경 변수 사용
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