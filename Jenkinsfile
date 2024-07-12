pipeline {
    environment { 
        DOCKER_ID = "ayakayu" 
        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"
        DOCKER_TAG = "latest" 
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
    }
    agent any
    options {
        timeout(time: 1, unit: 'HOURS')  // Set a timeout for the pipeline execution
        buildDiscarder(logRotator(numToKeepStr: '10'))  // Discard old builds to save space
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    def branch = env.GIT_BRANCH ?: 'master'
                    echo "Current branch is: ${branch}"
                    checkout scm
                }
            }
        }
        stage('Docker Build Images') {
            parallel {
                stage('Build Cast Service') {
                    steps {
                        script {
                            sh '''
                                docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
                            '''
                        }
                    }
                }
                stage('Build Movie Service') {
                    steps {
                        script {
                            sh '''
                                docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
                            '''
                        }
                    }
                }
            }
        }
        stage('Docker Push DockerHub') {
            parallel {
                stage('Push Cast Service') {
                    steps {
                        script {
                            sh '''
                                docker login -u $DOCKER_ID -p $DOCKER_PASS
                                docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                            '''
                        }
                    }
                }
                stage('Push Movie Service') {
                    steps {
                        script {
                            sh '''
                                docker login -u $DOCKER_ID -p $DOCKER_PASS
                                docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                            '''
                        }
                    }
                }
            }
        }
        stage('Deploy to Environments') {
            matrix {
                axes {
                    axis {
                        name 'ENV'
                        values 'dev', 'qa', 'staging', 'prod'
                    }
                }
                stages {
                    stage('Deploy') {
                        steps {
                            script {
                                if (env.ENV == 'prod') {
                                    input message: "Deploy to ${ENV}?", ok: "Deploy" 
                                }
                                sh '''
                                    rm -Rf .kube
                                    mkdir .kube
                                    cat $KUBECONFIG > .kube/config

                                    # Ensure the namespace is clean
                                    kubectl delete namespace ${ENV} --ignore-not-found
                                    kubectl create namespace ${ENV}
                                    kubectl config set-context --current --namespace=${ENV}
                                    
                                    # Delete all resources in the namespace
                                    kubectl delete all --all --namespace=${ENV} || true
                                    sleep 10
                                    
                                    # Ensure all PVCs are deleted or patched
                                    if kubectl get pvc movie-db-pvc --namespace=${ENV}; then
                                        kubectl patch pvc movie-db-pvc --namespace=${ENV} -p '{"metadata":{"finalizers":null}}' || true
                                        kubectl delete pvc movie-db-pvc --namespace=${ENV} --ignore-not-found || true
                                    fi
                                    
                                    if kubectl get pvc cast-db-pvc --namespace=${ENV}; then
                                        kubectl patch pvc cast-db-pvc --namespace=${ENV} -p '{"metadata":{"finalizers":null}}' || true
                                        kubectl delete pvc cast-db-pvc --namespace=${ENV} --ignore-not-found || true
                                    fi

                                    # Confirm PVCs are deleted
                                    kubectl get pvc --namespace=${ENV}

                                    # Get PV names and delete them
                                    pv_names=$(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="${ENV}")].metadata.name}')
                                    if [ -n "$pv_names" ]; then
                                        for pv in $pv_names; do
                                            kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                                            kubectl delete pv $pv --ignore-not-found || true
                                        done
                                    else
                                        echo "No PVs found for namespace ${ENV}."
                                    fi

                                    # Fix PVC metadata if it exists
                                    if kubectl get pvc movie-db-pvc --namespace=${ENV}; then
                                        kubectl annotate pvc movie-db-pvc meta.helm.sh/release-name=movie-db-${ENV} --overwrite
                                        kubectl annotate pvc movie-db-pvc meta.helm.sh/release-namespace=${ENV} --overwrite
                                    fi

                                    # Fix any conflicting PVC ownership annotations
                                    if kubectl get pvc movie-db-pvc --namespace=${ENV}; then
                                        kubectl patch pvc movie-db-pvc --namespace=${ENV} -p '{"metadata":{"annotations":{"meta.helm.sh/release-name":"movie-db-${ENV}","meta.helm.sh/release-namespace":"${ENV}"}}}'
                                    fi

                                    # Deploy the Helm charts
                                    helm upgrade --install cast-db-${ENV} helm-exam/ --values=helm-exam/values.yaml --namespace ${ENV}
                                    helm upgrade --install movie-db-${ENV} helm-exam/ --values=helm-exam/values.yaml --namespace ${ENV}
                                '''
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'target/test-*.xml', allowEmptyArchive: true
            junit 'target/test-*.xml'
        }
        failure {
            emailext to: 'yumotoayaka@gmail.com',
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed."
        }
    }
}
