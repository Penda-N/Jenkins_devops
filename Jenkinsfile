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
                    def servicePorts = [:]

                    services.each { service ->
                        def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                        sh """
                            docker rm -f ${service} || true
                            docker run -d -P --name ${service} ${DOCKER_IMAGE}
                        """
                        def portMapping = sh(script: "docker port ${service}", returnStdout: true).trim()
                        def port = portMapping.split(':')[-1]
                        servicePorts[service] = port
                        echo "${service} is running on port ${port}"
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
                    services.each { serviceName, endpoint ->
                        def port = servicePorts[serviceName]
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
                            dir(service) {
                                sh """
                                    rm -Rf .kube
                                    mkdir -p .kube
                                    cat \$KUBECONFIG > .kube/config
                                    cp helm/${service}/values.yaml values.yaml
                                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yaml
                                    helm upgrade --install ${service} helm/${service} --values=values.yaml --namespace ${env}
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
                }
            }
        }
    }
}
