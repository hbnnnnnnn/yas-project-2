pipeline {
    agent any

    environment {
        DOCKER_HUB_USER  = 'hbnnn'
        DOCKER_CREDS_ID  = 'dockerhub-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_ID   = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.BRANCH_TAG  = env.BRANCH_NAME.replaceAll('/', '-')
                    echo "Commit: ${env.COMMIT_ID}  Branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Detect changed services') {
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

                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD 2>/dev/null || echo ""',
                        returnStdout: true
                    ).trim()

                    def toBuild = []

                    if (changedFiles == '') {
                        echo "No previous commit to diff — building all services"
                        toBuild = serviceMap.collect { k, v -> "${k}:${v}" }
                    } else {
                        serviceMap.each { svcName, svcPath ->
                            if (changedFiles.split('\n').any { it.startsWith("${svcPath}/") }) {
                                toBuild << "${svcName}:${svcPath}"
                                echo "Changed: ${svcName}"
                            }
                        }
                        if (toBuild.isEmpty()) {
                            echo "No service folders changed — skipping build"
                        }
                    }

                    env.SERVICES_TO_BUILD = toBuild.join('\n')
                }
            }
        }

        stage('Build & Push images') {
            when {
                expression { env.SERVICES_TO_BUILD?.trim() != '' }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.trim().split('\n')

                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDS_ID) {
                        services.each { entry ->
                            def parts   = entry.split(':')
                            def svcName = parts[0]
                            def svcPath = parts[1]
                            def image   = "${env.DOCKER_HUB_USER}/${svcName}"
                            def context = "./${svcPath}"

                            echo "Building ${image}:${env.COMMIT_ID} from ${context}"

                            def img = docker.build("${image}:${env.COMMIT_ID}", context)

                            img.push(env.COMMIT_ID)
                            img.push(env.BRANCH_TAG)

                            echo "Pushed ${image}:${env.COMMIT_ID} + :${env.BRANCH_TAG}"
                        }
                    }
                }
            }
        }
    }

    post {
        success { echo "CI done. Tag: ${env.COMMIT_ID}" }
        failure { echo "CI failed — see logs above." }
    }
}