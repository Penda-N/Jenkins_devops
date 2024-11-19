pipeline {
    environment {
        DOCKER_ID = "pendand" // Docker ID
        DOCKER_TAG = "v.${BUILD_ID}.0" // Tag basé sur le numéro de build
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
                    def ports = ['8081', '8082'] // Ports distincts pour éviter les conflits
                    for (int i = 0; i < services.size(); i++) {
                        def service = services[i]
                        def port = ports[i]
                        def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                        sh """
                            docker rm -f ${service} || true
                            docker run -d -p ${port}:80 --name ${service} ${DOCKER_IMAGE}
                        """
                    }
                }
            }
        }
stage('Test Acceptance') {
    steps {
        script {
            def services = [
                ['port': '8081', 'endpoint': '/api/v1/casts/docs'],
                ['port': '8082', 'endpoint': '/api/v1/movies/docs']
            ]
            services.each { service ->
                // Attente que le service soit prêt
                def retryCount = 0
                def maxRetries = 10
                def serviceReady = false
                while (retryCount < maxRetries && !serviceReady) {
                    def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${service.port}${service.endpoint}", returnStdout: true).trim()
                    if (response == '200') {
                        serviceReady = true
                        echo "Service ${service.endpoint} is ready"
                    } else {
                        retryCount++
                        echo "Waiting for ${service.endpoint} to be ready (Attempt ${retryCount}/${maxRetries})"
                        sleep 5 // Attendre 5 secondes avant de réessayer
                    }
                }

                // Si après les tentatives le service n'est toujours pas prêt, échouer le test
                if (!serviceReady) {
                    error "Service ${service.endpoint} did not become ready after ${maxRetries} attempts"
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
    }
}
