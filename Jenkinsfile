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
                        dir(service) {
                            sh """
                                docker build -t ${DOCKER_ID}/${service}:${DOCKER_TAG} .
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
                    services.each { service ->
                        sh """
                            docker rm -f ${service} || true
                            docker run -d -p 8080:80 --name ${service} ${DOCKER_ID}/${service}:${DOCKER_TAG}
                        """
                    }
                }
            }
        }
        stage('Test Acceptance') {
            steps {
                script {
                    sh """
                        curl -s http://localhost:8080 || exit 1
                    """
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
                        sh """
                            docker push ${DOCKER_ID}/${service}:${DOCKER_TAG}
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
