pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'hbnnn'
        DOCKER_CREDS_ID = 'dockerhub-creds'
    }

    parameters {
        booleanParam(name: 'BUILD_ALL', defaultValue: false, description: 'Force build all services')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.COMMIT_ID  = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.BRANCH_TAG = env.BRANCH_NAME.replaceAll('/', '-')
                    echo "Commit: ${env.COMMIT_ID}  Branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Detect changed services') {
            steps {
                script {
                    def javaServices = [
                        'media'    : 'media',
                        'product'  : 'product',
                        'cart'     : 'cart',
                        'order'    : 'order',
                        'rating'   : 'rating',
                        'customer' : 'customer',
                        'location' : 'location',
                        'inventory': 'inventory',
                        'tax'      : 'tax',
                        'search'   : 'search',
                        'payment'        : 'payment',
                        'payment-paypal' : 'payment-paypal',
                        'promotion': 'promotion',
                        'recommendation': 'recommendation',
                        'webhook'  : 'webhook',
                        'storefront-bff': 'storefront-bff',
                        'backoffice-bff': 'backoffice-bff',
                        'sampledata': 'sampledata'
                    ]
                    def nodeServices = [
                        'storefront': 'storefront',
                        'backoffice': 'backoffice',
                    ]
                    def javaBuilds = []
                    def nodeBuilds = []

                    if (params.BUILD_ALL) {
                        echo "BUILD_ALL=true → building all services"
                        javaBuilds = javaServices.collect { k, v -> "${k}:${v}" }
                        nodeBuilds = nodeServices.collect { k, v -> "${k}:${v}" }

                    } else {
                        def changedFiles = sh(
                            script: 'git diff --name-only HEAD~1 HEAD 2>/dev/null || echo ""',
                            returnStdout: true
                        ).trim()

                        if (changedFiles == '') {
                            echo "No previous commit to diff — building all services"
                            javaBuilds = javaServices.collect { k, v -> "${k}:${v}" }
                            nodeBuilds = nodeServices.collect { k, v -> "${k}:${v}" }
                        } else {
                            javaServices.each { svcName, svcPath ->
                                if (changedFiles.split('\n').any { it.startsWith("${svcPath}/") }) {
                                    javaBuilds << "${svcName}:${svcPath}"
                                    echo "Changed (Java): ${svcName}"
                                }
                            }
                            nodeServices.each { svcName, svcPath ->
                                if (changedFiles.split('\n').any { it.startsWith("${svcPath}/") }) {
                                    nodeBuilds << "${svcName}:${svcPath}"
                                    echo "Changed (Node): ${svcName}"
                                }
                            }
                            if (javaBuilds.isEmpty() && nodeBuilds.isEmpty()) {
                                echo "No service folders changed — skipping build"
                            }
                        }
                    }
                    env.JAVA_SERVICES = javaBuilds.join('\n')
                    env.NODE_SERVICES = nodeBuilds.join('\n')
                    env.BUILD_REQUIRED = (!javaBuilds.isEmpty() || !nodeBuilds.isEmpty()).toString()
                }
            }
        }

        stage('Maven build') {
            when {
                expression { env.BUILD_REQUIRED == 'true' }
            }
            steps {
                script {
                    echo "Building full Maven reactor"
                    sh "mvn package -DskipTests --no-transfer-progress"
                }
            }
        }

        stage('Build & Push images') {
            when {
                expression {
                    (env.JAVA_SERVICES?.trim() != '') || (env.NODE_SERVICES?.trim() != '')
                }
            }
            steps {
                script {
                    def allServices = []
                    if (env.JAVA_SERVICES?.trim()) {
                        allServices += env.JAVA_SERVICES.trim().split('\n').toList()
                    }
                    if (env.NODE_SERVICES?.trim()) {
                        allServices += env.NODE_SERVICES.trim().split('\n').toList()
                    }

                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDS_ID) {
                        allServices.each { entry ->
                            def parts   = entry.split(':')
                            def svcName = parts[0]
                            def svcPath = parts[1]
                            def image   = "${env.DOCKER_HUB_USER}/${svcName}"

                            if (fileExists("${svcPath}/Dockerfile")) {
                                echo "Building ${image}:${env.COMMIT_ID} from ./${svcPath}"
                                def img = docker.build("${image}:${env.COMMIT_ID}", "./${svcPath}")
                                img.push(env.COMMIT_ID)
                                img.push(env.BRANCH_TAG)
                                echo "Pushed ${image}:${env.COMMIT_ID} + :${env.BRANCH_TAG}"
                            } else {
                                echo "Skipping image build for ${svcName}: missing ${svcPath}/Dockerfile"
                            }
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