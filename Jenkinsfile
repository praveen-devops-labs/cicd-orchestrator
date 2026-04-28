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

    // triggers { cron('H/30 * * * *') }
    triggers {
        githubPush()
    }

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

                    dir('env-control') {
                        deleteDir()

                        git url: 'https://github.com/praveen-devops-labs/env-control-repo.git',
                            branch: 'main',
                            credentialsId: 'github-token'
                    }

                    def config = readYaml file: 'env-control/mappings/current.yaml'

                    if (!config?.mappings) {
                        error "❌ mappings not found in YAML"
                    }

                    env.MAP = groovy.json.JsonOutput.toJson(config.mappings)

                    echo "📄 Mapping loaded:"
                    echo env.MAP
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

                            def commit

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
                                    script: "git rev-parse --short HEAD",
                                    returnStdout: true
                                ).trim()
                            }

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