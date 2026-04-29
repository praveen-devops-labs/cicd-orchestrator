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
                    gchatNotify(
                                "Orchestrator Started" 
                            )
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

        stage('Load Notifications') {
            steps {
                script {

                    def notifConfig = readYaml file: 'env-control/mappings/notifications.yaml'

                    if (!notifConfig?.notifications) {
                        error "❌ notifications missing"
                    }

                    env.NOTIFY_MAP = groovy.json.JsonOutput.toJson(notifConfig.notifications)

                    echo "📢 Notification mapping loaded"
                }
            }
        }    

        // 🔷 2. Process
        stage('Process') {
            steps {
                script {

                    def notifyMap = readJSON text: env.NOTIFY_MAP
                    def parsed = readJSON text: env.MAP

                    def jobs = [:]

                    parsed.each { key, value ->

                        // ✅ define first
                        def app     = value.app.toString()
                        def repo    = value.repo.toString()
                        def branch  = value.branch.toString()
                        def envName = value.env.toString()

                        jobs[key] = {

                            echo "🚀 " + app + " | " + branch + " → " + envName

                            // ✅ NOW safe to use
                            def credentialId = notifyMap[app] != null ? notifyMap[app][envName] : null

                            if (credentialId == null) {
                                credentialId = notifyMap['default']
                            }

                            echo "Using webhook: " + credentialId

                            // (for now just log, don’t use it yet)

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

                            echo "Latest commit: " + commit

                            def buildState = "/u01/jenkins/jenkins/build-state/" + app + "/" + envName + ".commit"

                            def lastBuilt = sh(
                                script: "[ -f " + buildState + " ] && cat " + buildState + " || echo none",
                                returnStdout: true
                            ).trim()

                            if (commit == lastBuilt) {
                                echo "⏭️ Already built → skipping"
                                return
                            }

                            gchatNotify(
                                "[" + app + "] Triggering " + branch + " → " + envName
                            )

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
                                        string(name: 'COMMIT_FULL', value: commit),
                                        string(name: 'VERSION', value: "latest")
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