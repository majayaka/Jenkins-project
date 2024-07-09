
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

stage('Deploiement en dev'){
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/db/values/cast-db/values-dev.yaml values.yml
                cat values.yml
                helm upgrade --install cast-db-dev manifests/db/ --values=values.yml --namespace dev
                
                cp manifests/db/values/movie-db/values-dev.yaml values.yml
                cat values.yml
                helm upgrade --install movie-db-dev manifests/db/ --values=values.yml --namespace dev
                '''
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/service/values/cast-service/values-dev.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-dev manifests/service/ --values=values.yml --namespace dev
                
                cp manifests/service/values/movie-service/values-dev.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-dev manifests/service/ --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en staging'){
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/db/values/cast-db/values-staging.yaml values.yml
                cat values.yml
                helm upgrade --install cast-db-staging manifests/db/ --values=values.yml --namespace staging
                
                cp manifests/db/values/movie-db/values-staging.yaml values.yml
                cat values.yml
                helm upgrade --install movie-db-staging manifests/db/ --values=values.yml --namespace staging
                '''
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/service/values/cast-service/values-staging.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-staging manifests/service/ --values=values.yml --namespace staging
                
                cp manifests/service/values/movie-service/values-staging.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-staging manifests/service/ --values=values.yml --namespace staging
                '''
                }
            }

        }

stage('Deploiement en QA'){
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/db/values/cast-db/values-qa.yaml values.yml
                cat values.yml
                helm upgrade --install cast-db-qa manifests/db/ --values=values.yml --namespace qa
                
                cp manifests/db/values/movie-db/values-qa.yaml values.yml
                cat values.yml
                helm upgrade --install movie-db-qa manifests/db/ --values=values.yml --namespace qa
                '''
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/service/values/cast-service/values-qa.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-qa manifests/service/ --values=values.yml --namespace qa
                
                cp manifests/service/values/movie-service/values-qa.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-qa manifests/service/ --values=values.yml --namespace qa
                '''
                }
            }
        }
  stage('Deploiement en prod'){
        when {
            expression { env.GIT_BRANCH == 'origin/master'}
        }
            steps {
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Voulez-vous déployer en production ?', ok: 'Oui'
                    }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/db/values/cast-db/values-prod.yaml values.yml
                cat values.yml
                helm upgrade --install cast-db-prod manifests/db/  --values=values.yml --namespace prod
                
                cp manifests/db/values/movie-db/values-prod.yaml values.yml
                cat values.yml
                helm upgrade --install movie-db-prod manifests/db/ --values=values.yml --namespace prod
                '''
                }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp manifests/service/values/cast-service/values-prod.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install cast-service-prod manifests/service/ --values=values.yml --namespace prod
                
                cp manifests/service/values/movie-service/values-prod.yaml values.yml
                sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                cat values.yml
                helm upgrade --install movie-service-prod manifests/service/ --values=values.yml --namespace prod
                '''
                }
            }

        }


}
}