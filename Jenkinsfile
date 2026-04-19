pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'hbnnn'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'   // credential ID set in Jenkins UI
        GIT_COMMIT_ID = ''
    }

    // All YAS backend services + frontend(s)
    // Map: service param name -> path inside the repo
    // Add/remove as needed to match your fork
    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_ID = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    echo "Commit ID: ${env.GIT_COMMIT_ID}"
                    echo "Branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Detect changed services') {
            steps {
                script {
                    // Compare current commit against its parent to find changed dirs
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD || git diff --name-only HEAD',
                        returnStdout: true
                    ).trim().split('\n')

                    // Map of service name -> subfolder in repo
                    def serviceMap = [
                        'media'     : 'media',
                        'product'   : 'product',
                        'cart'      : 'cart',
                        'order'     : 'order',
                        'rating'    : 'rating',
                        'customer'  : 'customer',
                        'location'  : 'location',
                        'inventory' : 'inventory',
                        'tax'       : 'tax',
                        'search'    : 'search',
                        'storefront': 'storefront',   // Next.js frontend
                        'backoffice': 'backoffice',   // Next.js backoffice
                    ]

                    def servicesToBuild = []

                    serviceMap.each { svcName, svcPath ->
                        def affected = changedFiles.any { f -> f.startsWith("${svcPath}/") }
                        if (affected) {
                            servicesToBuild << [name: svcName, path: svcPath]
                            echo "Will build: ${svcName}"
                        }
                    }

                    // If nothing changed (e.g. first commit), build everything
                    if (servicesToBuild.isEmpty()) {
                        echo "No specific service changed — building all services"
                        servicesToBuild = serviceMap.collect { k, v -> [name: k, path: v] }
                    }

                    env.SERVICES_JSON = groovy.json.JsonOutput.toJson(servicesToBuild)
                }
            }
        }

        stage('Build & Push images') {
            steps {
                script {
                    def services = new groovy.json.JsonSlurperClassic()
                        .parseText(env.SERVICES_JSON)

                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS_ID) {
                        services.each { svc ->
                            def imageName = "${env.DOCKER_HUB_USER}/${svc.name}"
                            def tag       = env.GIT_COMMIT_ID
                            def context   = "./${svc.path}"

                            echo "Building ${imageName}:${tag} from ${context}"

                            // Build image
                            def img = docker.build("${imageName}:${tag}", context)

                            // Push commit-id tag
                            img.push(tag)

                            // Also update branch-named tag so latest of branch is trackable
                            def branchTag = env.BRANCH_NAME.replaceAll('/', '-')
                            img.push(branchTag)

                            echo "Pushed ${imageName}:${tag} and ${imageName}:${branchTag}"
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "CI complete. Images pushed with tag: ${env.GIT_COMMIT_ID}"
        }
        failure {
            echo "CI failed. Check logs above."
        }
    }
}