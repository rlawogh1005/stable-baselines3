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

    stages {
        stage('Setup Environment') {
            steps {
                script {
                    checkout scm // .env 파일을 읽기 위해 먼저 git checkout이 필요합니다.
                    if (fileExists('.env')) {
                        echo ">>> [SETUP] Reading .env file and injecting to environment..."
                        def props = readProperties file: '.env'
                        props.each { key, value ->
                            env."${key}" = value
                            echo "Successfully loaded: ${key}"
                        }
                    } else {
                        echo ">>> [WARNING] .env file not found. Falling back to Jenkins environment."
                    }
                }
            }
        }

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
        // Stage 2: Sync Hierarchical Data to Users Module
        // ================================================================
        stage('Sync CI Hierarchical Data') {
            steps {
                script {
                    echo ">>> [SYNC] Fetching Collaborators for ${env.REPO_OWNER}/${env.REPO_NAME}..."
                    def repoOwner = env.REPO_OWNER
                    def repoName = env.REPO_NAME
                    
                    def collaboratorsResponse = sh(
                        script: """
                            curl -s -H "Authorization: token ${env.GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${repoOwner}/${repoName}/collaborators
                        """,
                        returnStdout: true
                    ).trim()
                    
                    def collaboratorsJson = []
                    if (collaboratorsResponse && collaboratorsResponse.startsWith('[')) {
                        def rawCollabs = readJSON text: collaboratorsResponse
                        // Map to CreateUserDto[] format
                        rawCollabs.each { collab ->
                            collaboratorsJson.add([
                                githubUid: collab.id,
                                username: collab.login,
                                email: collab.email ?: "${collab.login}@github.dummy.com"
                            ])
                        }
                    } else {
                        echo ">>> Warning: Failed to fetch collaborators or empty. Response: ${collaboratorsResponse}"
                    }

                    echo ">>> Merging Hierarchical Payload..."
                    def treesitterAst = readJSON file: 'ast_treesitter.json'
                    def commitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()

                    def payload = [
                        users: collaboratorsJson,
                        teamProject: [
                            teamName: "stable-baselines3",
                            jenkinsJobName: env.JOB_NAME,
                            sonarProjectKey: env.SONAR_PROJECT_KEY
                        ],
                        buildReport: [
                            buildNumber: env.BUILD_NUMBER.toInteger(),
                            buildUrl: env.BUILD_URL ?: "UNKNOWN_URL",
                            status: qualityGateResult ? qualityGateResult.status : "UNKNOWN",
                            commitHash: commitHash
                        ],
                        astNodes: treesitterAst.nodes
                    ]
                    
                    writeJSON file: 'payload_hierarchical.json', json: payload

                    echo ">>> Sending Hierarchical Payload to Users Module..."
                    def syncStatus = sh(
                        script: """
                            curl -s -o /dev/null -w '%{http_code}' \
                            -X POST "${env.SWV_BACKEND_URL}/users/sync-ci" \
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