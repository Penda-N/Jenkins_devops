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
        stage('Docker Push') {
            steps {
                script {
                    def services = ['cast-service', 'movie-service']

                    withCredentials([string(credentialsId: 'DOCKER_HUB_PASS', variable: 'DOCKER_PASS')]) {
                        sh """
                            echo "${DOCKER_PASS}" | docker login -u ${DOCKER_ID} --password-stdin
                        """
                        services.each { service ->
                            def DOCKER_IMAGE = "${DOCKER_ID}/${service}:${DOCKER_TAG}"
                            echo "Pushing Docker image for ${service}..."
                            sh """
                                docker push ${DOCKER_IMAGE}
                            """
                        }
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
                            echo "Deploying ${service} to ${env} environment"
                            def k8sPath = "k8s/${service}"

                            dir(k8sPath) {
                                sh """
                                    mkdir -p ~/.kube
                                    echo \$KUBECONFIG > ~/.kube/config
                                    chmod 600 ~/.kube/config

                                    sed -i "s+image:.*+image: ${DOCKER_ID}/${service}:${DOCKER_TAG}+g" ${service}-deployment.yaml

                                    kubectl apply -n ${env} -f ${service}-deployment.yaml
                                    kubectl apply -n ${env} -f ${service}-service.yaml

                                    kubectl rollout status deployment/${service} -n ${env} || exit 1
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
