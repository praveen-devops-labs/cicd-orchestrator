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

        stage('Load Mapping') {
            steps {
                script {
                    dir('env-control') {
                        deleteDir()

                        git url: env.CONTROL_REPO,
                            branch: 'main',
                            credentialsId: 'github-token'
                    }

                    // ✅ Direct YAML read (NO sandbox issue)
                    def config = readYaml file: 'env-control/mappings/current.yaml'

                    if (!config?.mappings) {
                        error "❌ mappings not found in YAML"
                    }

                    // store as normal map (IMPORTANT: no serialization issues)
                    env.MAPPING_KEYS = config.mappings.keySet().join(",")

                    // store entire object in memory (not env)
                    currentBuild.description = "Mappings loaded"
                    // we will reuse config in same script block later
                    // (don't try to store full map in env → causes LazyMap issue)

                    // save to global variable safely
                    // using Jenkins trick
                    binding.setVariable("MAPPINGS_OBJ", config.mappings)
                }
            }
        }

        stage('Process') {
            steps {
                script {

                    def mapping = binding.getVariable("MAPPINGS_OBJ")

                    def jobs = [:]

                    mapping.each { key, value ->

                        jobs[key] = {

                            def app     = value.app
                            def repo    = value.repo
                            def branch  = value.branch
                            def envName = value.env

                            echo "🚀 ${app} | ${branch} → ${envName}"

                            // 🔷 1. Get latest commit safely
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

                            // 🔥 3. Skip if already built
                            if (latestCommit == lastBuilt) {
                                echo "⏭️ Skipping ${branch} (already built)"
                                return
                            }

                            notify("🚀 Triggering ${app} ${branch}")

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
        success { echo "✅ Orchestrator success" }
        failure { echo "❌ Orchestrator failed" }
    }
}