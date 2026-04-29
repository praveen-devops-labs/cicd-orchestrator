@Library('cicd-library') _

pipeline {
    agent any

    triggers {
        cron('H/2 * * * *')   // 🔥 every 2 mins (good for testing)
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

        // 🔷 Start
        stage('Start') {
            steps {
                script {
                    gchatNotify("🚀 Orchestrator started")
                }
            }
        }

        // 🔷 Load Mapping
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

                    // ✅ Safe serialization
                    env.MAP = groovy.json.JsonOutput.toJson(config.mappings)

                    echo "📄 Mapping loaded"
                }
            }
        }

        // 🔷 Process (Polling Scan)
        stage('Process') {
            steps {
                script {

                    def parsed = readJSON text: env.MAP
                    def jobs = [:]
                    def changesDetected = false

                    parsed.each { key, value ->

                        // ✅ extract primitives (NO LazyMap issues)
                        def app     = value.app.toString()
                        def repo    = value.repo.toString()
                        def branch  = value.branch.toString()
                        def envName = value.env.toString()

                        // 🔥 ONLY build for dev
                        if (envName != 'dev') {
                            return
                        }

                        // 🔷 Lightweight commit fetch (NO clone)
                        def commit

                        withCredentials([usernamePassword(
                            credentialsId: 'github-token',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_PASS'
                        )]) {

                            def safeRepo = repo.replace("https://", "https://${GIT_USER}:${GIT_PASS}@")

                            commit = sh(
                                script: "git ls-remote ${safeRepo} refs/heads/${branch} | cut -f1",
                                returnStdout: true
                            ).trim()
                        }

                        if (!commit) {
                            echo "⚠️ Cannot fetch commit for ${app}"
                            return
                        }

                        // 🔷 State file
                        def stateFile = "/u01/jenkins/jenkins/build-state/${app}/dev.commit"

                        def lastBuilt = sh(
                            script: "[ -f ${stateFile} ] && cat ${stateFile} || echo none",
                            returnStdout: true
                        ).trim()

                        echo "🔍 ${app} | ${branch}"
                        echo "   Latest : ${commit}"
                        echo "   Last   : ${lastBuilt}"

                        // 🔥 Skip if no change
                        if (commit == lastBuilt) {
                            echo "⏭️ No changes → skipping ${app}"
                            return
                        }

                        // 🔥 Mark change detected
                        changesDetected = true

                        // 🔷 Parallel job
                        jobs[key] = {

                            echo "🚀 Triggering ${app} (${branch})"

                            gchatNotify("🚀 ${app} | ${branch} → dev")

                            build job: 'Build-Pipeline',
                                wait: false,
                                parameters: [
                                    string(name: 'APP_NAME', value: app),
                                    string(name: 'REPO_URL', value: repo),
                                    string(name: 'BRANCH_NAME', value: branch)
                                ]
                        }
                    }

                    // 🔥 Exit fast if no changes
                    if (!changesDetected) {
                        echo "⏭️ No changes across all repos"
                        currentBuild.description = "No changes"
                        return
                    }

                    // 🔥 Run parallel builds
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