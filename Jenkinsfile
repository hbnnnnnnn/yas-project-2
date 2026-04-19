pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'hbnnn'
        DOCKER_CREDS_ID = 'dockerhub-creds'
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
                    ]
                    def nodeServices = [
                        'storefront': 'storefront',
                        'backoffice': 'backoffice',
                    ]

                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD 2>/dev/null || echo ""',
                        returnStdout: true
                    ).trim()

                    def javaBuilds = []
                    def nodeBuilds = []

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

                    env.JAVA_SERVICES = javaBuilds.join('\n')
                    env.NODE_SERVICES = nodeBuilds.join('\n')
                }
            }
        }

        // Install common-library + parent pom into local Maven repo
        // so individual service builds can resolve internal dependencies
        stage('Install common dependencies') {
            when {
                expression { env.JAVA_SERVICES?.trim() != '' }
            }
            steps {
                sh '''
                    mvn install -DskipTests --no-transfer-progress \
                        --projects common-library \
                        --also-make
                '''
            }
        }

        stage('Maven build') {
            when {
                expression { env.JAVA_SERVICES?.trim() != '' }
            }
            steps {
                script {
                    env.JAVA_SERVICES.trim().split('\n').each { entry ->
                        def parts   = entry.split(':')
                        def svcName = parts[0]
                        def svcPath = parts[1]

                        echo "Running mvn package for ${svcName}"
                        sh """
                            cd ${svcPath}
                            mvn package -DskipTests --no-transfer-progress
                        """
                    }
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

                            echo "Building ${image}:${env.COMMIT_ID} from ./${svcPath}"
                            def img = docker.build("${image}:${env.COMMIT_ID}", "./${svcPath}")
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