@Library('cicd-library') _

pipeline {
    agent any

    triggers { 
        githubPush()        
    }

    // triggers { 
    //     githubPush()
    //     cron('H/30 * * * *') 
    // }

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
                    gchatNotify("🚀 Orchestrator started", "platform", "dev")
                }
            }
        }

        // 🔷 1. Load Mapping
        stage('Load Mapping') {
            steps {
                script {
                    
                    dir('env-control') {
                        deleteDir()

                        git url: CONTROL_REPO,
                            branch: 'main',
                            credentialsId: 'github-token'
                    }

                    def config = readYaml file: 'env-control/mappings/current.yaml'

                    if (!config?.mappings) {
                        error "❌ mappings missing in YAML"
                    }

                    // ✅ SAFE serialization
                    env.MAP = groovy.json.JsonOutput.toJson(config.mappings)

                    echo "📄 Mapping loaded:"
                    echo env.MAP
                }
            }
        }

        // 🔷 2. Process
        stage('Process') {
            steps {
                script {

                    // ✅ SAFE parse
                    def parsed = readJSON text: env.MAP

                    def jobs = [:]

                    parsed.each { key, value ->

                        // ✅ extract primitives (IMPORTANT)
                        def app     = "${value.app}"
                        def repo    = "${value.repo}"
                        def branch  = "${value.branch}"
                        def envName = "${value.env}"

                        jobs[key] = {

                            echo "🚀 ${app} | ${branch} → ${envName}"

                            def commit

                            // 🔷 Get latest commit
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

                                commit = sh(
                                    script: "git rev-parse HEAD",
                                    returnStdout: true
                                ).trim()
                            }

                            echo "Latest commit: ${commit}"

                            // 🔥 FIXED PATH
                            def buildState = "/u01/jenkins/jenkins/build-state/${app}/${envName}.commit"

                            def lastBuilt = sh(
                                script: "[ -f ${buildState} ] && cat ${buildState} || echo none",
                                returnStdout: true
                            ).trim()

                            echo "Last built: ${lastBuilt}"

                            // 🔥 CRITICAL CHECK
                            if (commit == lastBuilt) {
                                echo "⏭️ Already built → skipping"
                                return
                            }

                            gchatNotify(
                                "Triggering ${app} | ${branch} → ${envName}",
                                app,
                                envName
                            )

                            // 🔷 Trigger
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