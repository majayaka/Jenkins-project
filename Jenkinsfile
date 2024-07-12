pipeline {
    environment { 
        DOCKER_ID = "ayakayu" 
        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"
        DOCKER_TAG = "latest" 
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        KUBECONFIG = "/tmp/kubeconfig"
    }
    agent any
    stages {
        stage('Docker Build Image') {
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
        stage('Prepare Kubernetes Config') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        sh '''
                            rm -Rf .kube
                            mkdir .kube
                            cp $KUBECONFIG_FILE $KUBECONFIG
                        '''
                    }
                }
            }
        }
        stage('Clean Up dev-env') {
            steps {
                script {
                    sh '''
                        # Ensure the namespace is clean
                        kubectl delete namespace dev --ignore-not-found
                        kubectl create namespace dev
                        kubectl config set-context --current --namespace=dev

                        # Ensure all PVCs are deleted or patched
                        if kubectl get pvc movie-db-pvc --namespace=dev; then
                            kubectl patch pvc movie-db-pvc --namespace=dev -p '{"metadata":{"finalizers":null}}' || true
                            kubectl delete pvc movie-db-pvc --namespace=dev --ignore-not-found || true
                        fi
                        
                        if kubectl get pvc cast-db-pvc --namespace=dev; then
                            kubectl patch pvc cast-db-pvc --namespace=dev -p '{"metadata":{"finalizers":null}}' || true
                            kubectl delete pvc cast-db-pvc --namespace=dev --ignore-not-found || true
                        fi

                        # Confirm PVCs are deleted
                        kubectl get pvc --namespace=dev

                        # Get PV names and delete them
                        pv_names=$(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="dev")].metadata.name}')
                        if [ -n "$pv_names" ]; then
                            for pv in $pv_names; do
                                kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pv $pv --ignore-not-found || true
                            done
                        else
                            echo "No PVs found for namespace dev."
                        fi
                    '''
                }
            }
        }
        stage('Deploy to dev-env') {
            steps {
                script {
                    sh '''
                        # Check if the release exists
                        if helm status dev-env --namespace dev >/dev/null 2>&1; then
                            # Release exists, perform an upgrade
                            helm upgrade --install dev-env ./helm-exam --namespace dev --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG}
                        else
                            # Release does not exist, perform an installation
                            helm install dev-env ./helm-exam --namespace dev --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG}
                        fi
                    '''
                }
            }
        }
        stage('Deploy to qa-env') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        sh '''
                            # Ensure the namespace is clean
                            kubectl delete namespace qa --ignore-not-found
                            kubectl create namespace qa
                            kubectl config set-context --current --namespace=qa

                            # Ensure all PVCs are deleted or patched
                            if kubectl get pvc movie-db-pvc --namespace=qa; then
                                kubectl patch pvc movie-db-pvc --namespace=qa -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pvc movie-db-pvc --namespace=qa --ignore-not-found || true
                            fi
                            
                            if kubectl get pvc cast-db-pvc --namespace=qa; then
                                kubectl patch pvc cast-db-pvc --namespace=qa -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pvc cast-db-pvc --namespace=qa --ignore-not-found || true
                            fi

                            # Confirm PVCs are deleted
                            kubectl get pvc --namespace=qa

                            # Get PV names and delete them
                            pv_names=$(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="qa")].metadata.name}')
                            if [ -n "$pv_names"; then
                                for pv in $pv_names; do
                                    kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                                    kubectl delete pv $pv --ignore-not-found || true
                                done
                            else
                                echo "No PVs found for namespace qa."
                            fi

                            # Check if the release exists
                            if helm status qa-env --namespace qa >/dev/null 2>&1; then
                                # Release exists, perform an upgrade
                                helm upgrade --install qa-env ./helm-exam --namespace qa --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG} --set namespace=qa --set ingress.host=qa.datascientest-landes.cloudns.ch
                            else
                                # Release does not exist, perform an installation
                                helm install qa-env ./helm-exam --namespace qa --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG} --set namespace=qa --set ingress.host=qa.datascientest-landes.cloudns.ch
                            fi
                        '''
                    }
                }
            }
        }
        stage('Deploy to staging-env') {
            steps {
                input message: 'Do you want to deploy to staging?', ok: 'Deploy'
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        sh '''
                            # Ensure the namespace is clean
                            kubectl delete namespace staging --ignore-not-found
                            kubectl create namespace staging
                            kubectl config set-context --current --namespace=staging

                            # Ensure all PVCs are deleted or patched
                            if kubectl get pvc movie-db-pvc --namespace=staging; then
                                kubectl patch pvc movie-db-pvc --namespace=staging -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pvc movie-db-pvc --namespace=staging --ignore-not-found || true
                            fi
                            
                            if kubectl get pvc cast-db-pvc --namespace=staging; then
                                kubectl patch pvc cast-db-pvc --namespace=staging -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pvc cast-db-pvc --namespace=staging --ignore-not-found || true
                            fi

                            # Confirm PVCs are deleted
                            kubectl get pvc --namespace=staging

                            # Get PV names and delete them
                            pv_names=$(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="staging")].metadata.name}')
                            if [ -n "$pv_names"; then
                                for pv in $pv_names; do
                                    kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                                    kubectl delete pv $pv --ignore-not-found || true
                                done
                            else
                                echo "No PVs found for namespace staging."
                            fi

                            # Check if the release exists
                            if helm status staging-env --namespace staging >/dev/null 2>&1; then
                                # Release exists, perform an upgrade
                                helm upgrade --install staging-env ./helm-exam --namespace staging --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG} --set namespace=staging --set ingress.host=staging.datascientest-landes.cloudns.ch
                            else
                                # Release does not exist, perform an installation
                                helm install staging-env ./helm-exam --namespace staging --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG} --set namespace=staging --set ingress.host=staging.datascientest-landes.cloudns.ch
                            fi
                        '''
                    }
                }
            }
        }
        stage('Deploy to production') {
            steps {
                input message: 'Do you want to deploy to production?', ok: 'Deploy'
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        sh '''
                            # Ensure the namespace is clean
                            kubectl delete namespace prod --ignore-not-found
                            kubectl create namespace prod
                            kubectl config set-context --current --namespace=prod

                            # Ensure all PVCs are deleted or patched
                            if kubectl get pvc movie-db-pvc --namespace=prod; then
                                kubectl patch pvc movie-db-pvc --namespace=prod -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pvc movie-db-pvc --namespace=prod --ignore-not-found || true
                            fi
                            
                            if kubectl get pvc cast-db-pvc --namespace=prod; then
                                kubectl patch pvc cast-db-pvc --namespace=prod -p '{"metadata":{"finalizers":null}}' || true
                                kubectl delete pvc cast-db-pvc --namespace=prod --ignore-not-found || true
                            fi

                            # Confirm PVCs are deleted
                            kubectl get pvc --namespace=prod

                            # Get PV names and delete them
                            pv_names=$(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="prod")].metadata.name}')
                            if [ -n "$pv_names"; then
                                for pv in $pv_names; do
                                    kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                                    kubectl delete pv $pv --ignore-not-found || true
                                done
                            else
                                echo "No PVs found for namespace prod."
                            fi

                            # Check if the release exists
                            if helm status production-env --namespace prod >/dev/null 2>&1; then
                                # Release exists, perform an upgrade
                                helm upgrade --install production-env ./helm-exam --namespace prod --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG} --set namespace=prod --set ingress.host=prod.datascientest-landes.cloudns.ch
                            else
                                # Release does not exist, perform an installation
                                helm install production-env ./helm-exam --namespace prod --set castService.image.tag=${DOCKER_TAG},movieService.image.tag=${DOCKER_TAG} --set namespace=prod --set ingress.host=prod.datascientest-landes.cloudns.ch
                            fi
                        '''
                    }
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
