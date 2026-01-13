def qualityGateResult

properties([
    parameters([
        string(name: 'SONAR_PROJECT_KEY', defaultValue: 'your-python-project-key', description: 'SonarQube Project Key'),
        string(name: 'SWV_BACKEND_URL', defaultValue: 'http://mp-backend:3000/api/team-projects', description: 'SWV Backend Notification URL'),
        string(name: 'PYEXAMINE_URL', defaultValue: 'http://pyexamine-service:8000/analyze', description: 'PyExamine Service URL')
    ])
])

pipeline {
    agent {
        docker {
            image 'python:3.10'
            // [권장] 루트 권한으로 실행하여 apt-get 설치 허용
            args '-u root:root --network=shared-net'
        }
    }

    environment {
        SONAR_SERVER       = 'SonarQube-Server'
        SONAR_CREDENTIALS  = 'SONAR_QUBE_TOKEN'
        SWV_CREDENTIALS    = 'SWV_BACKEND_TOKEN_ID'
        PYTHONIOENCODING   = 'utf-8'
    }
    
    // [수정 1] 도구 이름을 Jenkins 에러 로그에 나온 제안대로 변경
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
                    // [수정 2] sonar-scanner 실행을 위해 Java(JRE) 필수 설치
                    sh 'apt-get update && apt-get install -y openjdk-17-jre zip curl'
                    
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
                        curl -X POST "${params.PYEXAMINE_URL}" \
                        -F "file=@source_code.zip" \
                        -H "accept: application/json"
                    """, returnStdout: true).trim()

                    echo ">>> PyExamine Analysis Completed."
                    
                    def backendMetricsUrl = "${params.SWV_BACKEND_URL}/metrics"
                    
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
                        // [확인] Tool 설정이 올바르면 'sonar-scanner' 명령어가 PATH에 자동 등록됨
                        sh """
                            sonar-scanner \
                            -Dsonar.projectKey=${params.SONAR_PROJECT_KEY} \
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
                            url: params.SWV_BACKEND_URL,
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

        stage('Install Dependencies') {
            steps {
                script {
                    // [수정] openjdk-17-jre 대신 default-jre 사용
                    // default-jre는 해당 리눅스 버전에서 가장 안정적인 Java 버전을 자동으로 설치합니다.
                    sh 'apt-get update && apt-get install -y default-jre zip curl'
                    
                    if (fileExists('requirements.txt')) {
                        sh 'pip install -r requirements.txt'
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