pipeline {
    environment { 
        DOCKER_ID = "ayakayu" 
        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"
        DOCKER_TAG = "latest" 
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
    }
    agent any
    stages {
        stage('Docker Build') { 
            steps {
                script {
                    sh '''
                    docker rmi -f ayakayu/cast-service
                    docker rmi -f ayakayu/movie-service
                    docker build -t ayakayu/cast-service ./cast-service
                    docker build -t ayakayu/movie-service ./movie-service
                    sleep 6
                    '''
                }
            }
        }
        stage('Docker Push') { 
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
        stage('Deployment in dev') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {  // 変更箇所
                    script {
                        sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cp $KUBECONFIG .kube/config
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
                        cp $KUBECONFIG .kube/config
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
        }
        stage('Deployment in staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {  // 変更箇所
                    script {
                        sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cp $KUBECONFIG .kube/config
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
                        cp $KUBECONFIG .kube/config
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
        }
        stage('Deployment in QA') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {  // 変更箇所
                    script {
                        sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cp $KUBECONFIG .kube/config
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
                        cp $KUBECONFIG .kube/config
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
        }
        stage('Deployment in prod') {
            when {
                expression { env.GIT_BRANCH == 'origin/master' }
            }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {  // 変更箇所
                    script {
                        sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cp $KUBECONFIG .kube/config
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
                        cp $KUBECONFIG .kube/config
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
}
