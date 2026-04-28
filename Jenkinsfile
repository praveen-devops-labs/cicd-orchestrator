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

        stage('Discover') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${APP_REPO} | awk '{print \$2}' | sed 's#refs/heads/##'",
                        returnStdout: true
                    ).trim().split("\n")

                    env.BRANCHES = branches.join(",")
                }
            }
        }

        stage('Process') {
            steps {
                script {

                    def mapping = new groovy.json.JsonSlurper().parseText(env.MAP)
                    def jobs = [:]

                    env.BRANCHES.split(",").each { b ->

                        if (!mapping.containsKey(b)) return

                        def envName = mapping[b]

                        jobs[b] = {

                            def commit = sh(
                                script: "git ls-remote ${APP_REPO} refs/heads/${b} | cut -f1 | cut -c1-7",
                                returnStdout: true
                            ).trim()

                            def file = "/opt/deploy-state/${APP_NAME}/${envName}.commit"
                            def exists = sh(script: "[ -f ${file} ] && cat ${file} || echo none", returnStdout: true).trim()

                            if (commit == exists) {
                                echo "⏭️ Skip ${b}"
                                return
                            }

                            if (envName == 'dev') {

                                build job: 'Build-Pipeline',
                                    wait: false,
                                    parameters: [
                                        string(name: 'APP_NAME', value: APP_NAME),
                                        string(name: 'REPO_URL', value: APP_REPO),
                                        string(name: 'BRANCH_NAME', value: b)
                                    ]

                            } else {

                                build job: 'Deploy-Pipeline',
                                    wait: false,
                                    parameters: [
                                        string(name: 'APP_NAME', value: APP_NAME),
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