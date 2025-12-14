pipeline {
    environment {
        DOCKER_ID = "alberttyty"
        DOCKER_IMG_MOVIE = "movie-service"
        DOCKER_IMG_CAST = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"

        KUBE_CONF = credentials('kubectl-config')

        APP_PORT = 30007
    }
    agent any
    stages {
        stage('Docker build') {
            steps {
                script {
                    sh '''
                        docker build -t $DOCKER_ID/$DOCKER_IMG_MOVIE:$DOCKER_TAG ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMG_CAST:$DOCKER_TAG ./cast-service
                    '''
                }
            }
        }
        stage('Docker push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }
            steps {
                script {
                    sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$DOCKER_IMG_MOVIE:$DOCKER_TAG
                        docker push $DOCKER_ID/$DOCKER_IMG_CAST:$DOCKER_TAG
                    '''
                }
            }
        }
        stage('Dev deployments') {
            steps {
                script {
                    def envs = ['dev', 'qa', 'staging']
                    def SERVICES = ['movie', 'cast']
                    for (int i = 0; i < envs.size(); ++i) {
                        def ENV_NAME = envs[i]
                        for (int j = 0; j < SERVICES.size(); ++j) {
                            def SERVICE = SERVICES[j]
                            def SERVICE_PORT = (i + 1) * 100 + APP_PORT.toInteger() + j
                            def SERVICE_DB = "--set db.name=${SERVICE}"
                            sh """
                                rm -Rf .kube
                                mkdir .kube
                                cat \$KUBE_CONF > .kube/config
                                export KUBECONFIG=.kube/config
                                cp charts/values.yaml values.yml
                                sed -i "s+repository.*+repository: \$DOCKER_ID/${SERVICE}-service+g" values.yml
                                sed -i "s+tag.*+tag: \$DOCKER_TAG+g" values.yml
                                
                                sed -i "s+nodePort.*+nodePort: ${SERVICE_PORT}+g" values.yml

                                helm upgrade --install ${SERVICE}-app charts --values=values.yml ${SERVICE_DB} --namespace ${ENV_NAME}
                            """
                        }
                    }
                }
            }
        }
        stage('Prod deployment') {
            when {
                branch 'master'
            }
            steps {
                input message: 'Proceed to production deployment ?'
                script {
                    def SERVICES = ['movie', 'cast']
                    for (int i = 0; i < SERVICES.size(); ++i) {
                        def SERVICE = SERVICES[i]
                        def SERVICE_DB = "--set db.name=${SERVICE}"
                        sh """
                            rm -Rf .kube
                            mkdir .kube
                            cat \$KUBE_CONF > .kube/config
                            export KUBECONFIG=.kube/config
                            cp charts/values.yaml values.yml
                            sed -i "s+repository.*+repository: \$DOCKER_ID/${SERVICE}+g" values.yml
                            sed -i "s+tag.*+tag: \$DOCKER_TAG+g" values.yml
                            
                            sed -i "s+nodePort.*+nodePort: ${APP_PORT + i}+g" values.yml
                            
                            helm upgrade --install ${SERVICE}-app charts --values=values.yml ${SERVICE_DB} --namespace prod --set image.repository=\$DOCKER_ID/${SERVICE}-service --set image.tag=\$DOCKER_TAG
                        """
                    }
                }
            }
        }
    }
}