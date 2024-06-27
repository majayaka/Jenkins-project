pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "ndiayepi" // replace this with your docker-id
DOCKER_IMAGE = "cast-service-image"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any // Jenkins will be able to select all available agents 
stages {
        stage('Initialization'){ // Prepare environment
            steps {
                script {
                sh '''
                 sudo chmod 644 /etc/rancher/k3s/k3s.yaml

                 sleep 3
                '''
                }
            }
        }
        stage('Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                 ./remove_container_if_exists.sh cast_service
                 ./remove_container_if_exists.sh cast_db
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG cast-service
                sleep 6
                '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''
                    docker-compose -f db-docker-compose.yml up -d
                    sleep 10
                    docker-compose -f services-docker-compose.yml up -d
                    sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl  -X GET -i http://localhost:8002/api/v1/casts/docs
                    sleep 5
                    docker-compose -f services-docker-compose.yml stop
                    docker-compose -f db-docker-compose.yml stop
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                '''
                }
            }

        }

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp cast-service-helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-service-helm --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp cast-service-helm/values.yaml values.yml 
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-service-helm --values=values.yml --namespace staging
                '''
                }
            }

        }
stage('Deploiement en QA'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp cast-service-helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-service-helm --values=values.yml --namespace qa
                '''
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh ''' 
                rm -Rf .kube 
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp cast-service-helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-service-helm --values=values.yml --namespace prod
                '''
                }
            }
        }
}
post { // send email when the job has failed
    // ..
    failure {
        echo "This will run if the job failed"
        mail to: "pierrendiaye@gmail.com",
             subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
             body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
    }
    // ..
}
}