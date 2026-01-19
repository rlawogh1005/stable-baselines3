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
        PARSER_URL='http://mp-parser:3001/analyze'
        SONAR_PROJECT_KEY='stable-baselines3'
        SONAR_SERVER='SonarQube-Server'
        SONAR_CREDENTIALS='SONAR_QUBE_TOKEN'
        PYTHONIOENCODING='utf-8'
    }
    
    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner-Latest'
    }

    stages {
        stage('Initialize & Analysis') {
            steps {
                script {
                    checkout scm
                    sh 'apt-get update && apt-get install -y zip curl default-jre'
                    sh 'git config --global --add safe.directory "*"'

                    echo ">>> Zipping Source Code..."
                    sh 'zip -r code_package.zip . -x "*.git*" "node_modules/*" "dist/*" "__pycache__/*"'

                    echo ">>> Sending to Parser Container..."
                    def parserResponse = sh(
                        script: "curl -s -X POST '${env.PARSER_URL}' -F 'file=@code_package.zip'", 
                        returnStdout: true
                    ).trim()
                    
                    // [최적화] 단순 문자열 contains("error") 제거
                    // JSON으로 파싱하여 실제 데이터 구조를 검증
                    def parserJson = readJSON text: parserResponse
                    
                    if (!parserJson.nodes) {
                        error "Parser Error: No nodes found in response. Raw: ${parserResponse}"
                    }

                    writeFile file: 'ast_result.json', text: parserResponse
                    echo ">>> AST Data verified and saved. (Nodes: ${parserJson.nodes.size()})"
                }
            }
        }

        stage('SonarQube & Quality Gate') {
            steps {
                script {
                    sh 'apt-get update && apt-get install -y default-jre zip curl'
                    sh 'git config --global --add safe.directory "*"'
                    if (fileExists('requirements.txt')) {
                        sh 'pip install -r requirements.txt'
                    }
                    
                    def scannerHome = tool 'SonarScanner-Latest'
                    withSonarQubeEnv(env.SONAR_SERVER) {
                        sh """
                            "${scannerHome}/bin/sonar-scanner" \
                            -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.language=py \
                            -Dsonar.python.version=3.10
                        """
                    }
                    qualityGateResult = waitForQualityGate abortPipeline: true, credentialsId: env.SONAR_CREDENTIALS
                }
            }   
        }

        stage('PyExamine & Integrated Report') {
            steps {
                script {
                    echo ">>> Starting PyExamine Analysis..."
                    def pyExamineResponse = sh(
                        script: "curl -s -X POST '${env.PYEXAMINE_URL}' -F 'file=@code_package.zip'", 
                        returnStdout: true
                    ).trim()

                    def rawSmellResults = readJSON text: pyExamineResponse
                    def rawAstResults = readJSON file: 'ast_result.json'
                    
                    // [핵심] 모든 데이터를 하나로 결합
                    def mergedPayload = [
                        teamName: "stable-baselines3", 
                        jenkinsJobName: env.JOB_NAME,
                        sonarProjectKey: env.SONAR_PROJECT_KEY,
                        analysis: [
                            jobName: env.JOB_NAME,
                            buildNumber: env.BUILD_NUMBER.toInteger(),
                            status: qualityGateResult.status, // SonarQube 결과 반영
                            buildUrl: env.BUILD_URL,
                            commitHash: sh(returnStdout: true, script: 'git rev-parse HEAD').trim(),
                            pyExamineResult: rawSmellResults,
                            astResults: rawAstResults.nodes // AST 데이터 포함
                        ],
                        astResult: rawAstResults
                    ]
                    echo ">>> Merged Payload: ${mergedPayload}"
                    writeJSON file: 'final_payload.json', json: mergedPayload
                    
                    echo ">>> Sending Integrated Payload to Backend..."
                    sh """
                        curl -X POST "${env.SWV_BACKEND_URL}/team-projects" \
                        -H "Content-Type: application/json" \
                        -d @final_payload.json
                    """
                }
            }
        }
    }

    post {
        always {
            // 모든 임시 파일 삭제
            sh 'rm -f *.zip *.json'
            cleanWs()
        }
    }
}