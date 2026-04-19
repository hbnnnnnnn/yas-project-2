pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        DOCKER_HUB_USER = 'hbnnn'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'   // credential ID set in Jenkins UI
        GIT_COMMIT_ID = ''
        SERVICES_TO_BUILD = ''
    }

    // All YAS backend services + frontend(s)
    // Map: service param name -> path inside the repo
    // Add/remove as needed to match your fork
    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def commitId = sh(
                        script: 'git rev-parse --short HEAD || echo latest',
                        returnStdout: true
                    ).trim()

                    def resolvedCommitId = (commitId ?: '').trim()
                    if (!resolvedCommitId) {
                        resolvedCommitId = 'latest'
                    }

                    env.GIT_COMMIT_ID = resolvedCommitId
                    writeFile file: '.ci_commit_id', text: "${resolvedCommitId}\n"

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
                            servicesToBuild << svcName
                            echo "Will build: ${svcName}"
                        }
                    }

                    // If nothing changed (e.g. first commit), build everything
                    if (servicesToBuild.isEmpty()) {
                        echo "No specific service changed — building all services"
                        servicesToBuild = serviceMap.collect { k, v -> k }
                    }

                    def servicesCsv = servicesToBuild.join(',')
                    env.SERVICES_TO_BUILD = servicesCsv
                    writeFile file: '.ci_services_to_build', text: "${servicesCsv}\n"
                    echo "Services selected: ${servicesCsv}"
                }
            }
        }

        stage('Build service artifacts') {
            steps {
                script {
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
                        'storefront': 'storefront',
                        'backoffice': 'backoffice',
                    ]

                    def servicesText = ''
                    if (fileExists('.ci_services_to_build')) {
                        servicesText = readFile('.ci_services_to_build').trim()
                    }
                    if (!servicesText && env.SERVICES_TO_BUILD?.trim()) {
                        servicesText = env.SERVICES_TO_BUILD.trim()
                    }

                    def selectedServices = servicesText
                        ? servicesText.split(',').collect { it.trim() }.findAll { it }
                        : (serviceMap.keySet() as List)

                    selectedServices.each { svcName ->
                        if (!serviceMap.containsKey(svcName)) {
                            echo "Skipping unknown service key: ${svcName}"
                            return
                        }

                        def svcPath = serviceMap[svcName]

                        // Java services need the jar in target/ before docker build COPY.
                        if (fileExists("${svcPath}/pom.xml")) {
                            dir(svcPath) {
                                if (fileExists('mvnw')) {
                                    sh './mvnw -B -DskipTests package'
                                } else {
                                    sh 'mvn -B -DskipTests package'
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Build & Push images') {
            steps {
                script {
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
                        'storefront': 'storefront',
                        'backoffice': 'backoffice',
                    ]

                    def servicesText = ''
                    if (fileExists('.ci_services_to_build')) {
                        servicesText = readFile('.ci_services_to_build').trim()
                    }
                    if (!servicesText && env.SERVICES_TO_BUILD?.trim()) {
                        servicesText = env.SERVICES_TO_BUILD.trim()
                    }

                    def selectedServices = servicesText
                        ? servicesText.split(',').collect { it.trim() }.findAll { it }
                        : (serviceMap.keySet() as List)

                    def commitTag = env.GIT_COMMIT_ID?.trim()
                    if (!commitTag && fileExists('.ci_commit_id')) {
                        commitTag = readFile('.ci_commit_id').trim()
                    }
                    if (!commitTag) {
                        commitTag = 'latest'
                    }

                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS_ID) {
                        selectedServices.each { svcName ->
                            if (!serviceMap.containsKey(svcName)) {
                                echo "Skipping unknown service key: ${svcName}"
                                return
                            }

                            def svcPath   = serviceMap[svcName]
                            def imageName = "${env.DOCKER_HUB_USER}/${svcName}"
                            def tag       = commitTag
                            def context   = "./${svcPath}"

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