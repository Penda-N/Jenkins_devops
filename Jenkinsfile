pipeline {
    environment {
        DOCKER_ID = "pendand"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    agent any
    stages {
        stage('Docker Build') {
            steps {
                script {
                    def services = ['cast-service', 'movie-service']
                    services.each { service ->
                        def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                        dir(service) {
                            echo "Building Docker image for ${service}"
                            sh """
                                docker build -t ${DOCKER_IMAGE} .
                            """
                        }
                    }
                }
            }
        }
        stage('Docker Run') {
            steps {
                script {
                    def services = ['cast-service', 'movie-service']
                    def basePort = 8000
                    def servicePorts = [:]

                    services.eachWithIndex { service, index ->
                    def port = basePort + index // Attribue un port unique pour chaque service
                    def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                    echo "Running Docker container for ${service} on port ${port}"
                    sh """
                        docker rm -f ${service} || true
                        docker run -d -p ${port}:8000 --name ${service} ${DOCKER_IMAGE}
                    """
                    servicePorts[service] = port
                    echo "${service} is running on port ${port}"
                    }

                    // Enregistrer les ports dans une variable d'environnement pour les étapes suivantes
                    env.SERVICE_PORTS = servicePorts.collect { key, value -> "${key}:${value}" }.join(',')
                }
            }
        }
        stage('Test Container Health') {
            steps {
                script {
                    def services = ['cast-service', 'movie-service']
                    services.each { service ->
                        def port = servicePorts[service]
                        def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${port}", returnStdout: true).trim()
                        if (response != '200') {
                            error "Service ${service} did not start successfully, HTTP status: ${response}"
                        } else {
                            echo "${service} is healthy, HTTP status: ${response}"
                        }
                    }
                }
            }
        }
        stage('Test Acceptance') {
            steps {
                script {
                    def services = [
                        'cast-service': '/api/v1/casts/docs',
                        'movie-service': '/api/v1/movies/docs'
                    ]

                    def servicePorts = env.SERVICE_PORTS.tokenize(',').collectEntries {
                        def parts = it.split(':')
                        [(parts[0]): parts[1]]
                    }

                    services.each { serviceName, endpoint ->
                        def port = servicePorts[serviceName]
                        echo "Testing ${serviceName} on http://localhost:${port}${endpoint}"
                        def retryCount = 0
                        def maxRetries = 10
                        def serviceReady = false

                        while (retryCount < maxRetries && !serviceReady) {
                            def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${port}${endpoint}", returnStdout: true).trim()
                            if (response == '200') {
                                serviceReady = true
                                echo "Service ${endpoint} is ready"
                            } else {
                                retryCount++
                                echo "Waiting for ${endpoint} to be ready (Attempt ${retryCount}/${maxRetries})"
                                sleep 5
                            }
                        }

                        if (!serviceReady) {
                            error "Service ${endpoint} did not become ready after ${maxRetries} attempts"
                        }
                    }
                }
            }
        }
        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    def services = ['cast-service', 'movie-service']
                    sh "docker login -u ${DOCKER_ID} -p ${DOCKER_PASS}"
                    services.each { service ->
                        def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                        echo "Pushing Docker image for ${service}"
                        sh """
                            docker push ${DOCKER_IMAGE}
                        """
                    }
                }
            }
        }
        stage('Deployment') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    def environments = ['dev', 'qa', 'staging', 'prod']
                    def services = ['cast-service', 'movie-service']

                    environments.each { env ->
                        services.each { service ->
                            def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                            echo "Deploying ${service} to ${env} environment"
                            dir(service) {
                                sh """
                                    mkdir -p ~/.kube
                                    echo \$KUBECONFIG > ~/.kube/config
                                    cp helm/${service}/values.yaml values.yaml
                                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yaml
                                    helm upgrade --install ${service} helm/${service} --values=values.yaml --namespace ${env}
                                    kubectl get pods -n ${env} | grep ${service}
                                """
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        failure {
            echo "Build failed."
            mail to: "penda.ndiaye10@gmail.com",
                subject: "${env.JOB_NAME} - Build #${env.BUILD_ID} Failed",
                body: "Check the console output for details: ${env.BUILD_URL}"
        }
        always {
            script {
                def services = ['cast-service', 'movie-service']
                services.each { service ->
                    sh "docker rm -f ${service} || true"
                    sh "docker rmi ${DOCKER_ID}/${service}:${DOCKER_TAG} || true"
                }
            }
        }
    }
}
