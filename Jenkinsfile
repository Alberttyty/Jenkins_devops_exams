pipeline {
    environment {
        DOCKER_ID = "alberttyty"
        DOCKER_IMG_MOVIE = "movie-service"
        DOCKER_IMG_CAST = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        KUBE_CONF = credentials('kubectl-config')
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
                    def services = ['movie-service', 'cast-service']
                    for (env in envs) {
                        for (int i = 0; i < services.size(); ++i) {
                            sh """
                                rm -Rf .kube
                                mkdir .kube
                                cat \${KUBE_CONF} > .kube/config
                                cp charts/values.yaml values.yml
                                sed -i 's+repository.*+repository: \${DOCKER_ID}/\${services[i]}+g' values.yml
                                sed -i 's+tag.*+tag: \${DOCKER_TAG}+g' values.yml
                                
                                sed -i 's+nodePort.*+nodePort: \${30007 + i}+g' values.yml

                                helm upgrade --install \${services[i]}-app charts --values=values.yml --namespace \${env}
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
                    def services = ['movie-service', 'cast-service']
                    for (int i = 0; i < services.size(); ++i) {
                        sh """
                            rm -Rf .kube
                            mkdir .kube
                            cat \${KUBE_CONF} > .kube/config
                            cp charts/values.yaml values.yml
                            sed -i 's+repository.*+repository: \${DOCKER_ID}/\${services[i]}+g' values.yml
                            sed -i 's+tag.*+tag: \${DOCKER_TAG}+g' values.yml
                            
                            sed -i 's+nodePort.*+nodePort: \${30007 + i}+g' values.yml
                            
                            helm upgrade --install \${services[i]}-app charts --values=values.yml --namespace prod
                        """
                    }
                }
            }
        }
    }
}