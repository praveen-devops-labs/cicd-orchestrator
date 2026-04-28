pipeline {
    agent any

    triggers {
        cron('H/30 * * * *')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_REPO = "https://github.com/praveen-devops-labs/cicd-orchestrator.git"
        CONTROL_REPO = "https://github.com/praveen-devops-labs/env-control-repo.git"
    }

    stages {

        // 🔷 1. Discover branches
        stage('Discover Branches') {
            steps {
                script {
                    def branches = sh(
                        script: """
                        git ls-remote --heads ${APP_REPO} \
                        | awk '{print \$2}' \
                        | sed 's#refs/heads/##'
                        """,
                        returnStdout: true
                    ).trim().split("\n")

                    env.BRANCHES = branches.join(",")

                    echo "🌿 Discovered branches:"
                    branches.each { echo " - ${it}" }
                }
            }
        }

        // 🔷 2. Load mapping file
        stage('Load Mapping') {
            steps {
                script {

                    dir('env-control') {
                        deleteDir()

                        git url: CONTROL_REPO, branch: 'main'
                    }

                    def config = readYaml file: 'env-control/mappings/current.yaml'

                    if (!config?.mappings) {
                        error "❌ Invalid mapping file: 'mappings' missing"
                    }

                    env.MAPPING_JSON = groovy.json.JsonOutput.toJson(config.mappings)

                    echo "📄 Mapping loaded:"
                    echo "${env.MAPPING_JSON}"
                }
            }
        }

        // 🔷 3. Process branches
        stage('Process Branches') {
            steps {
                script {

                    def branches = env.BRANCHES.split(",")
                    def mapping = new groovy.json.JsonSlurper().parseText(env.MAPPING_JSON)

                    branches.each { branch ->

                        if (!mapping.containsKey(branch)) {
                            echo "⏭️ Skipping ${branch} (no mapping)"
                            return
                        }

                        def envName = mapping[branch]

                        echo """
===============================
Branch   : ${branch}
Env      : ${envName}
===============================
"""

                        // 🔥 TEMP: Only log (next step will trigger jobs)
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Orchestrator completed successfully"
        }
        failure {
            echo "❌ Orchestrator failed"
        }
    }
}
