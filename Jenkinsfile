def SERVICE_MAP = [
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

def BACKEND_SERVICES = [
    'media', 'product', 'cart', 'order', 'rating',
    'customer', 'location', 'inventory', 'tax', 'search'
]

def getShortCommitHash() {
    return sh(script: 'git rev-parse --short=8 HEAD || echo latest', returnStdout: true).trim()
}

pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        DOCKER_HUB_USER = 'hbnnn'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        GIT_COMMIT_ID = ''
        SERVICES_TO_BUILD = ''
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def resolvedCommitId = getShortCommitHash()
                    if (!resolvedCommitId) {
                        resolvedCommitId = 'latest'
                    }

                    env.GIT_COMMIT_ID = resolvedCommitId.toString()
                    writeFile file: '.ci_commit_id', text: "${resolvedCommitId}\n"

                    echo "Commit ID: ${resolvedCommitId}"
                    echo "Branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Detect changed services') {
            steps {
                script {
                    def baseBranch = env.CHANGE_TARGET ?: 'main'
                    def diffCommand = ''

                    if (env.BRANCH_NAME == 'main') {
                        def hasParent = sh(script: 'git rev-parse HEAD~1', returnStatus: true) == 0
                        diffCommand = hasParent ? 'git diff --name-only HEAD~1 HEAD' : "git show --name-only --pretty='' HEAD"
                    } else {
                        sh "git fetch origin ${baseBranch}:refs/remotes/origin/${baseBranch} --depth=10"
                        diffCommand = "git diff --name-only origin/${baseBranch} HEAD"
                    }

                    def changedFilesRaw = sh(script: diffCommand, returnStdout: true).trim()
                    def changedFiles = changedFilesRaw ? changedFilesRaw.readLines().findAll { it?.trim() } : []
                    def servicesToBuild = [] as LinkedHashSet
                    def isRootChanged = false

                    changedFiles.each { file ->
                        if (file == 'pom.xml' || file.startsWith('common-library/')) {
                            isRootChanged = true
                        }

                        def topLevelDir = file.tokenize('/').first()
                        if (topLevelDir && SERVICE_MAP.containsKey(topLevelDir)) {
                            servicesToBuild << topLevelDir
                        }
                    }

                    if (isRootChanged || servicesToBuild.isEmpty()) {
                        echo isRootChanged ? 'Root/common-library changed, building all services.' : 'No specific service changed, building all services.'
                        servicesToBuild = SERVICE_MAP.keySet() as LinkedHashSet
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
                    def servicesText = fileExists('.ci_services_to_build') ? readFile('.ci_services_to_build').trim() : env.SERVICES_TO_BUILD?.trim()
                    def selectedServices = servicesText
                        ? servicesText.split(',').collect { it.trim() }.findAll { it }
                        : (SERVICE_MAP.keySet() as List)

                    selectedServices.each { svcName ->
                        if (!SERVICE_MAP.containsKey(svcName)) {
                            echo "Skipping unknown service key: ${svcName}"
                            return
                        }

                        def svcPath = SERVICE_MAP[svcName]

                        if (BACKEND_SERVICES.contains(svcName) && fileExists("${svcPath}/pom.xml")) {
                            sh """
                                docker run --rm \
                                  -v \"${WORKSPACE}\":/workspace \
                                  -v \"${WORKSPACE}/.m2\":/root/.m2 \
                                  -w /workspace \
                                  maven:3.9-eclipse-temurin-25 \
                                  sh -lc 'if [ -f ${svcPath}/mvnw ]; then chmod +x ${svcPath}/mvnw || true; ${svcPath}/mvnw -f pom.xml -B -DskipTests -pl ${svcPath} -am install; else mvn -f pom.xml -B -DskipTests -pl ${svcPath} -am install; fi'
                            """
                        }
                    }
                }
            }
        }

        stage('Build & Push images') {
            steps {
                script {
                    def servicesText = fileExists('.ci_services_to_build') ? readFile('.ci_services_to_build').trim() : env.SERVICES_TO_BUILD?.trim()
                    def selectedServices = servicesText
                        ? servicesText.split(',').collect { it.trim() }.findAll { it }
                        : (SERVICE_MAP.keySet() as List)

                    def commitTag = env.GIT_COMMIT_ID?.trim()
                    if (!commitTag && fileExists('.ci_commit_id')) {
                        commitTag = readFile('.ci_commit_id').trim()
                    }
                    if (!commitTag) {
                        commitTag = 'latest'
                    }

                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS_ID) {
                        selectedServices.each { svcName ->
                            if (!SERVICE_MAP.containsKey(svcName)) {
                                echo "Skipping unknown service key: ${svcName}"
                                return
                            }

                            def svcPath = SERVICE_MAP[svcName]
                            def dockerfilePath = "${svcPath}/Dockerfile"

                            if (!fileExists(dockerfilePath)) {
                                echo "Skipping ${svcName}: ${dockerfilePath} not found"
                                return
                            }

                            def imageName = "${env.DOCKER_HUB_USER}/${svcName}"
                            def context = "./${svcPath}"
                            def branchTag = (env.BRANCH_NAME ?: 'branch').replaceAll('/', '-')

                            echo "Building ${imageName}:${commitTag} from ${context}"
                            def img = docker.build("${imageName}:${commitTag}", context)
                            img.push(commitTag)
                            img.push(branchTag)

                            echo "Pushed ${imageName}:${commitTag} and ${imageName}:${branchTag}"
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
            echo 'CI failed. Check logs above.'
        }
    }
}