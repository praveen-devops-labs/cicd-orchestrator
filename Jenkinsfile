@Library('cicd-library') _

pipeline {
    agent any

    triggers { 
        githubPush()
        cron('H/30 * * * *') 
    }

    options {
        disableConcurrentBuilds(abortPrevious: true)
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }

    environment {
        CONTROL_REPO = "https://github.com/praveen-devops-labs/env-control-repo.git"
    }

    stages {

        stage('Start') {
            steps {
                script {
                    notify("🚀 Orchestrator started")
                }
            }
        }

        // 🔷 1. Load Mapping from control repo
        stage('Load Mapping') {
            steps {
                script {
                    
                    dir('env-control') {
                        deleteDir()

                        git url: CONTROL_REPO,
                            branch: 'main',
                            credentialsId: 'github-token'
                    }

                    // Read YAML → convert to STRING → store in env (safe)
                    def config = readYaml file: 'env-control/mappings/current.yaml'

                    if (!config?.mappings) {
                        error "❌ mappings missing in YAML"
                    }

                    env.MAP = groovy.json.JsonOutput.toJson(config.mappings)

                    echo "📄 Mapping loaded:"
                    echo env.MAP
                }
            }
        }

        // 🔷 2. Process Mapping (SAFE)
        stage('Process') {
            steps {
                script {

                    def mapping = new groovy.json.JsonSlurperClassic()
                        .parseText(env.MAPPING_JSON)

                    def jobs = [:]

                    mapping.each { key, value ->

                        jobs[key] = {

                            def app    = value.app
                            def repo   = value.repo
                            def branch = value.branch
                            def envName= value.env

                            echo "🚀 ${app} | ${branch} → ${envName}"

                            // 🔷 1. Get latest commit
                            def latestCommit = ""

                            dir("tmp-${branch}") {
                                deleteDir()

                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: "*/${branch}"]],
                                    userRemoteConfigs: [[
                                        url: repo,
                                        credentialsId: 'github-token'
                                    ]]
                                ])

                                latestCommit = sh(
                                    script: "git rev-parse HEAD",
                                    returnStdout: true
                                ).trim()
                            }

                            echo "Latest commit: ${latestCommit}"

                            // 🔷 2. Read build-state
                            def stateFile = "/u01/jenkins/jenkins/build-state/${app}/dev.commit"

                            def lastBuilt = sh(
                                script: "[ -f ${stateFile} ] && cat ${stateFile} || echo none",
                                returnStdout: true
                            ).trim()

                            echo "Last built: ${lastBuilt}"

                            // 🔥 3. SKIP if same commit
                            if (latestCommit == lastBuilt) {
                                echo "⏭️ Skipping ${branch} (already built)"
                                return
                            }

                            // 🔷 4. Trigger build
                            build job: 'Build-Pipeline',
                                parameters: [
                                    string(name: 'APP_NAME', value: app),
                                    string(name: 'REPO_URL', value: repo),
                                    string(name: 'BRANCH_NAME', value: branch)
                                ],
                                wait: false
                        }
                    }

                    parallel jobs
                }
            }
        }
    }

    post {
        success {
            echo "✅ Orchestrator completed"
        }
        failure {
            echo "❌ Orchestrator failed"
        }
    }
}