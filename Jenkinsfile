def qualityGateResult

pipeline {
    agent {
        docker {
            image 'python:3.10'
            args '-u root:root --network=shared-net'
        }
    }

    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner-Latest'
    }

    environment {
        SWV_BACKEND_URL='http://codevi-backend:13000/api'
        TREESITTER_PARSER_URL='http://codevi-parser-treesitter:3001/analyze'
        
        REPO_OWNER = 'rlawogh1005'
        REPO_NAME = 'stable-baselines3'
        
        // 🛡️ 테스트용 임시 토큰 (발급받으신 ghp_... 를 여기에 직접 적어주세요)
        GITHUB_TOKEN = credentials('GITHUB_TOKEN_FOR_ANALYZE_PROJECT')
        
        PYTHONIOENCODING='utf-8'
    }

    stages {
        // stage('Setup Environment') {
        //     steps {
        //         script {
        //             checkout scm // .env 파일을 읽기 위해 먼저 git checkout이 필요합니다.
        //             if (fileExists('.env')) {
        //                 echo ">>> [SETUP] Reading .env file and injecting to environment..."
        //                 def props = readProperties file: '.env'
        //                 props.each { key, value ->
        //                     env."${key}" = value
        //                     echo "Successfully loaded: ${key}"
        //                 }
        //             } else {
        //                 echo ">>> [WARNING] .env file not found. Falling back to Jenkins environment."
        //             }
        //         }
        //     }
        // }

        // ================================================================
        // Stage 1: 소스코드 zip + Tree-sitter 파서 호출
        // ================================================================
        stage('Initialize & Parse') {
            steps {
                script {
                    checkout scm
                    sh 'apt-get update && apt-get install -y zip curl default-jre'
                    sh 'git config --global --add safe.directory "*"'

                    echo ">>> [INIT] Zipping codebase and sending to Parser at ${env.TREESITTER_PARSER_URL}..."
                    sh 'zip -r code_package.zip . -x "*.git*" "node_modules/*" "dist/*" "__pycache__/*"'

                    echo ">>> Sending to Tree-sitter Parser..."
                    def treesitterResponse = sh(
                        script: "curl -s -X POST '${TREESITTER_PARSER_URL}' -F 'file=@code_package.zip'",
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
        // Stage 2: Sync Hierarchical Data to Users Module
        // ================================================================
        stage('Sync CI Hierarchical Data') {
            steps {
                script {
                    echo ">>> [SYNC] Fetching Collaborators for ${REPO_OWNER}/${REPO_NAME}..."
                    def repoOwner = REPO_OWNER
                    def repoName = REPO_NAME
                    
                    def collaboratorsResponse = sh(
                        script: """
                            curl -s -H "Authorization: token \${GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${repoOwner}/${repoName}/collaborators
                        """,
                        returnStdout: true
                    ).trim()
                    
                    def collaboratorUsernames = []
                    def collaboratorUids = []
                    if (collaboratorsResponse && collaboratorsResponse.startsWith('[')) {
                        def rawCollabs = readJSON text: collaboratorsResponse
                        rawCollabs.each { collab ->
                            collaboratorUsernames.add(collab.login)
                            collaboratorUids.add(collab.id)
                        }
                    } else {
                        echo ">>> Warning: Failed to fetch collaborators or empty. Response: ${collaboratorsResponse}"
                    }

                    echo ">>> Merging Hierarchical Payload..."
                    def treesitterAst = readJSON file: 'ast_treesitter.json'

                    def payload = [
                        username: collaboratorUsernames,
                        githubUid: collaboratorUids,
                        ProjectDto: [
                            teamName: REPO_NAME,
                            jenkinsJobName: env.JOB_NAME,
                            sonarProjectKey: env.SONAR_PROJECT_KEY ?: "codevi:main",
                            analysis: [
                                buildNumber: env.BUILD_NUMBER.toInteger(),
                                buildUrl: env.BUILD_URL ?: "UNKNOWN_URL"
                            ],
                            astNodes: treesitterAst.nodes
                        ]
                    ]
                    
                    writeJSON file: 'payload_hierarchical.json', json: payload

                    echo ">>> Sending Hierarchical Payload to Users Module..."
                    def syncStatus = sh(
                        script: """
                            curl -s -o /dev/null -w '%{http_code}' \
                            -X POST "${env.SWV_BACKEND_URL}/users/build-report" \
                            -H "Content-Type: application/json" \
                            -d @payload_hierarchical.json
                        """,
                        returnStdout: true
                    ).trim()

                    if (syncStatus != '200' && syncStatus != '201') {
                        error "Users Module Sync responded with HTTP ${syncStatus}"
                    }
                    echo ">>> Users Module Sync accepted (HTTP ${syncStatus})"
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