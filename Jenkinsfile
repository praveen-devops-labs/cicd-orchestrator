def notify(msg) {
    def payload = groovy.json.JsonOutput.toJson([text: msg])
    writeFile file: 'gchat.json', text: payload

    withCredentials([string(credentialsId: 'gchat-webhook', variable: 'WEBHOOK')]) {
        sh '''
        curl -s -o /dev/null -X POST \
        -H "Content-Type: application/json" \
        --data @gchat.json "$WEBHOOK"
        '''
    }
}

pipeline {
    agent any

    triggers { cron('H/30 * * * *') }

    options {
        disableConcurrentBuilds(abortPrevious: true)
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }

    environment {
        APP_REPO = "https://github.com/praveen-devops-labs/jenkins-cicd-demo.git"
        APP_NAME = "myapp"
    }

    stages {

        stage('Load Mapping') {
            steps {
                script {
                    def config = readYaml file: 'mappings/current.yaml'
                    env.MAP = groovy.json.JsonOutput.toJson(config.mappings)
                }
            }
        }

        // stage('Discover') {
        //     steps {
        //         script {
        //             def branches = sh(
        //                 script: "git ls-remote --heads ${APP_REPO} | awk '{print \$2}' | sed 's#refs/heads/##'",
        //                 returnStdout: true
        //             ).trim().split("\n")

        //             env.BRANCHES = branches.join(",")
        //         }
        //     }
        // }

        stage('Process') {
            steps {
                script {

                    def mapping = new groovy.json.JsonSlurper().parseText(env.MAP)
                    def jobs = [:]

                    mapping.each { key, cfg ->

                        def app     = cfg.app
                        def repo    = cfg.repo
                        def branch  = cfg.branch
                        def envName = cfg.env

                        jobs[key] = {

                            echo "🚀 ${app} | ${branch} → ${envName}"

                            def commit = sh(
                                script: "git ls-remote ${repo} refs/heads/${branch} | cut -f1 | cut -c1-7",
                                returnStdout: true
                            ).trim()

                            def stateFile = "/opt/deploy-state/${app}/${envName}.commit"

                            def last = sh(
                                script: "[ -f ${stateFile} ] && cat ${stateFile} || echo none",
                                returnStdout: true
                            ).trim()

                            echo "Current: ${commit}, Last: ${last}"

                            if (commit == last) {
                                echo "⏭️ Skipping (no change)"
                                return
                            }

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
        success { notify("✅ Orchestrator Done") }
        failure { notify("❌ Orchestrator Failed") }
    }
}