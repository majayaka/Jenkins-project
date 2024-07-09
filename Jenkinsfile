
pipeline {
environment { // Declaration of environment variables
    DOCKER_ID = "ayakayu" 
    DOCKER_IMAGE_CAST = "cast-service"
    DOCKER_IMAGE_MOVIE = "movie-service"
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    KUBECONFIG = credentials("config") 
    
}
agent any
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
               
                script {
                sh '''
                 docker rm -f jenkins
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service/
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service/
                sleep 6
                '''
                }
            }
        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") 
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                '''
                }
            }

        }

stage('Deployment in dev'){
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install cast-db-dev helm-exam/ --values=values.yml --namespace dev
                
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install movie-db-dev helm-exam/ --values=values.yml --namespace dev
                '''
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-dev helm-exam/ --values=values.yml --namespace dev
                
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-dev helm-exam/ --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deployment in staging'){
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install cast-db-staging helm-exam/ --values=values.yml --namespace staging
                
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install movie-db-staging helm-exam/ --values=values.yml --namespace staging
                '''
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-staging helm-exam/ --values=values.yml --namespace staging
                
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-staging helm-exam/ --values=values.yml --namespace staging
                '''
                }
            }

        }

stage('Deployment in QA'){
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install cast-db-qa helm-exam/ --values=values.yml --namespace qa
                
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install movie-db-qa helm-exam/ --values=values.yml --namespace qa
                '''
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-qa helm-exam/ --values=values.yml --namespace qa
                
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-qa helm-exam/ --values=values.yml --namespace qa
                '''
                }
            }
        }

  stage('Deployment in prod'){
        when {
            expression { env.GIT_BRANCH == 'origin/master'}
        }
            steps {
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install cast-db-prod helm-exam/ --values=values.yml --namespace prod
                
                cp helm-exam/values.yml
                cat values.yml
                helm upgrade --install movie-db-prod helm-exam/ --values=values.yml --namespace prod
                '''
                }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp helm-exam/values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-prod helm-exam/ --values=values.yml --namespace prod
                
                cp manifests/service/values/movie-service/values-prod.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-prod helm-exam/ --values=values.yml --namespace prod
                '''
                }
            }

        }


}
}