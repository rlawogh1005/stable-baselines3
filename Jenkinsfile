def qualityGateResult

pipeline {
    agent {
        docker {
            image 'python:3.10'
            args '-u root:root --network=shared-net'
        }
    }

    environment {
        SWV_BACKEND_URL='http://codevi-backend:13000/api'
        RELATIONAL_BACKEND_URL='http://codevi-backend-relational:13001/api'
        JSON_BACKEND_URL='http://codevi-backend-json:13002/api'
        PYEXAMINE_URL='http://pyexamine-service:8000/analyze'
        TREESITTER_PARSER_URL='http://codevi-parser-treesitter:3001/analyze'
        SONAR_PROJECT_KEY='stable-baselines3'
        SONAR_SERVER='SonarQube-Server'
        SONAR_CREDENTIALS='SONAR_QUBE_TOKEN'
        PYTHONIOENCODING='utf-8'
    }
    
    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner-Latest'
    }

    stages {
        // ================================================================
        // Stage 1: 소스코드 zip + Tree-sitter 파서 호출
        // ================================================================
        stage('Initialize & Parse') {
            steps {
                script {
                    checkout scm
                    sh 'apt-get update && apt-get install -y zip curl default-jre'
                    sh 'git config --global --add safe.directory "*"'

                    echo ">>> Zipping Source Code..."
                    sh 'zip -r code_package.zip . -x "*.git*" "node_modules/*" "dist/*" "__pycache__/*"'

                    // ── Tree-sitter 파서 호출 (유일한 파서)
                    echo ">>> Sending to Tree-sitter Parser..."
                    def treesitterResponse = sh(
                        script: "curl -s -X POST '${env.TREESITTER_PARSER_URL}' -F 'file=@code_package.zip'",
                        returnStdout: true
                    ).trim()

                    def treesitterJson = readJSON text: treesitterResponse
                    if (!treesitterJson.nodes) {
                        error "TreeSitter Parser Error: No nodes found."
                    }

                    writeFile file: 'ast_treesitter.json', text: treesitterResponse
                    echo ">>> TreeSitter AST saved. (Nodes: ${treesitterJson.nodes.size()}, Benchmark: ${treesitterJson.benchmark})"
                }
            }
        }

        // ================================================================
        // Stage 2: Tree-sitter 데이터를 2개의 백엔드로 전송
        //   - V1 Relational (13001): 구조 단위 정규화 저장
        //   - V2 JSON       (13002): JSON 통째 저장
        // ================================================================
        stage('Send AST to Backends') {
            parallel {
                // ── V1: Relational AST 백엔드 (13001)
                stage('V1 - Relational AST') {
                    steps {
                        script {
                            echo ">>> Sending Tree-sitter AST to Relational Backend (V1:13001)..."
                            def treesitterAst = readJSON(file: 'ast_treesitter.json')
                            
                            def relationalPayload = [
                                jenkinsJobName: env.JOB_NAME,
                                nodes         : treesitterAst.nodes // 백엔드 DTO와 일치
                            ]
                            writeJSON file: 'payload_relational.json', json: relationalPayload

                            def relationalStatus = sh(
                                script: """
                                    curl -s -o /dev/null -w '%{http_code}' \
                                    -X POST "${env.RELATIONAL_BACKEND_URL}/ast-data/relational" \
                                    -H "Content-Type: application/json" \
                                    -d @payload_relational.json
                                """,
                                returnStdout: true
                            ).trim()

                            if (relationalStatus != '200' && relationalStatus != '201') {
                                error "Relational Backend responded with HTTP ${relationalStatus}"
                            }
                            echo ">>> Relational Backend accepted (HTTP ${relationalStatus})"
                        }
                    }
                }

                // ── V2: JSON AST 백엔드 (13002)
                stage('V2 - JSON AST') {
                    steps {
                        script {
                            echo ">>> Sending Tree-sitter AST to JSON Backend (V2:13002)..."
                            def treesitterAst = readJSON(file: 'ast_treesitter.json')

                            def jsonPayload = [
                                jenkinsJobName: env.JOB_NAME,
                                nodes         : treesitterAst.nodes // JSON 컬럼에 직렬화 저장됨
                            ]
                            writeJSON file: 'payload_json.json', json: jsonPayload

                            def jsonStatus = sh(
                                script: """
                                    curl -s -o /dev/null -w '%{http_code}' \
                                    -X POST "${env.JSON_BACKEND_URL}/ast-data/json" \
                                    -H "Content-Type: application/json" \
                                    -d @payload_json.json
                                """,
                                returnStdout: true
                            ).trim()

                            if (jsonStatus != '200' && jsonStatus != '201') {
                                error "JSON Backend responded with HTTP ${jsonStatus}"
                            }
                            echo ">>> JSON Backend accepted (HTTP ${jsonStatus})"
                        }
                    }
                }
            }
        }

        // ================================================================
        // Stage 3: SonarQube 분석 + Quality Gate
        // ================================================================
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

        // ================================================================
        // Stage 4: PyExamine + 통합 리포트 → 기존 백엔드(13000)
        // ================================================================
        // stage('PyExamine & Integrated Report') {
        //     steps {
        //         script {
        //             echo ">>> Starting PyExamine Analysis..."
        //             def pyExamineResponse = sh(
        //                 script: "curl -s -X POST '${env.PYEXAMINE_URL}' -F 'file=@code_package.zip'", 
        //                 returnStdout: true
        //             ).trim()

        //             def rawSmellResults = readJSON text: pyExamineResponse
        //             def treesitterAst = readJSON file: 'ast_treesitter.json'
                    
        //             // 통합 페이로드 (AST는 이미 V1/V2로 전송됨, 여기선 레퍼런스 수준으로 포함)
        //             def mergedPayload = [
        //                 teamName       : "stable-baselines3", 
        //                 jenkinsJobName : env.JOB_NAME,
        //                 sonarProjectKey: env.SONAR_PROJECT_KEY,
        //                 analysis: [
        //                     jobName        : env.JOB_NAME,
        //                     buildNumber    : env.BUILD_NUMBER.toInteger(),
        //                     status         : qualityGateResult.status,
        //                     buildUrl       : env.BUILD_URL,
        //                     commitHash     : sh(returnStdout: true, script: 'git rev-parse HEAD').trim(),
        //                     pyExamineResult: rawSmellResults,
        //                     astResults     : treesitterAst.nodes
        //                 ]
        //             ]
        //             echo ">>> Merged Payload ready."
        //             writeJSON file: 'final_payload.json', json: mergedPayload
                    
        //             echo ">>> Sending Integrated Payload to Backend..."
        //             sh """
        //                 curl -X POST "${env.SWV_BACKEND_URL}/team-projects" \
        //                 -H "Content-Type: application/json" \
        //                 -d @final_payload.json
        //             """
        //         }
        //     }
        // }
    }

    post {
        always {
            // 모든 임시 파일 삭제
            sh 'rm -f *.zip *.json'
            cleanWs()
        }
    }
}