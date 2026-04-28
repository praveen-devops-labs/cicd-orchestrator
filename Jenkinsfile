@Library('cicd-library') _

pipeline {
    agent any

    triggers { cron('H/30 * * * *') }

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

                    // 🔥 IMPORTANT: Convert JSON → pure serializable structure
                    def parsed = readJSON text: env.MAP

                    def jobs = [:]

                    parsed.each { key, value ->

                        // 🔥 Extract ONLY primitives (critical)
                        def app     = "${value.app}"
                        def repo    = "${value.repo}"
                        def branch  = "${value.branch}"
                        def envName = "${value.env}"

                        jobs[key] = {

                            echo "🚀 ${app} | ${branch} → ${envName}"

                            // 🔷 Get commit safely (no LazyMap, no credentials issues)
                            def commit

                            dir("tmp-${branch}") {
                                deleteDir()

                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: "${branch}"]],
                                    userRemoteConfigs: [[
                                        url: repo,
                                        credentialsId: 'github-token'
                                    ]]
                                ])

                                commit = sh(
                                    script: "git rev-parse --short HEAD",
                                    returnStdout: true
                                ).trim()
                            }
                            notify("🚀 ${app} | ${branch} → ${envName}")
                            // 🔷 Read last deployed commit
                            def buildState = "/opt/build-state/${app}/dev.commit"

                            def lastBuilt = sh(
                                script: "[ -f ${buildState} ] && cat ${buildState} || echo none",
                                returnStdout: true
                            ).trim()

                            if (commit == lastBuilt) {
                                echo "⏭️ Already built → skipping"
                                return
                            }

                            // 🔷 Trigger pipelines
                            if (envName == 'dev') {

                                build job: 'Build-Pipeline',
                                    wait: false,
                                    parameters: [
                                        string(name: 'APP_NAME', value: app),
                                        string(name: 'REPO_URL', value: repo),
                                        string(name: 'BRANCH_NAME', value: branch)
                                    ]

                            } else {

                                build job: 'Deploy-Pipeline',
                                    wait: false,
                                    parameters: [
                                        string(name: 'APP_NAME', value: app),
                                        string(name: 'TARGET_ENV', value: envName),
                                        string(name: 'COMMIT_HASH', value: commit)
                                    ]
                            }
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