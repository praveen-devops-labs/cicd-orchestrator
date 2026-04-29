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
                    gchatNotify("🚀 Orchestrator started")
                }
            }
        }

        stage('Load Mapping') {
            steps {
                script {
                    dir('env-control') {
                        deleteDir()
                        git url: CONTROL_REPO, branch: 'main', credentialsId: 'github-token'
                    }

                    def config = readYaml file: 'env-control/mappings/current.yaml'

                    if (!config?.mappings) {
                        error "❌ mappings missing"
                    }

                    env.MAP = groovy.json.JsonOutput.toJson(config.mappings)
                }
            }
        }

        stage('Process') {
            steps {
                script {

                    def parsed = readJSON text: env.MAP
                    def jobs = [:]

                    parsed.each { key, value ->

                        def app     = value.app.toString()
                        def repo    = value.repo.toString()
                        def branch  = value.branch.toString()
                        def envName = value.env.toString()

                        jobs[key] = {

                            echo "🚀 " + app + " | " + branch + " → " + envName

                            def commit

                            dir("tmp-" + branch) {
                                deleteDir()

                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: "*/" + branch]],
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

                            def stateFile = "/u01/jenkins/jenkins/build-state/" + app + "/" + envName + ".commit"

                            def lastBuilt = sh(
                                script: "[ -f " + stateFile + " ] && cat " + stateFile + " || echo none",
                                returnStdout: true
                            ).trim()

                            if (commit == lastBuilt) {
                                echo "⏭️ Skipping (no change)"
                                return
                            }

                            gchatNotify("[" + app + "] Trigger " + branch + " → " + envName)

                            if (envName == 'dev') {

                                build job: 'Build-Pipeline',
                                    wait: false,
                                    parameters: [
                                        string(name: 'APP_NAME', value: app),
                                        string(name: 'REPO_URL', value: repo),
                                        string(name: 'BRANCH_NAME', value: branch)
                                    ]

                            } else {

                                echo "⏭️ Skipping " + envName + " (handled via promotion pipeline)"

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