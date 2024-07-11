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
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Docker Build') { 
            steps {
                script {
                    sh '''
                    docker rmi -f $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG || true
                    docker rmi -f $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG || true
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
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
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {
                    script {
                        sh '''
                        # Copy kubeconfig file
                        cp $KUBECONFIG /tmp/kubeconfig
                        
                        # Configure the auth of file
                        chmod 600 /tmp/kubeconfig
                        
                        # Configure env
                        export KUBECONFIG=/tmp/kubeconfig
                        
                        # Ensure the namespace is clean
                        kubectl delete namespace dev --ignore-not-found
                        kubectl create namespace dev
                        kubectl config set-context --current --namespace=dev
                        
                        # Delete all resources in the namespace
                        kubectl delete all --all --namespace=dev || true
                        sleep 10
                        
                        # Ensure all PVCs are deleted
                        kubectl delete pvc movie-db-pvc --namespace=dev --ignore-not-found || true
                        kubectl delete pvc cast-db-pvc --namespace=dev --ignore-not-found || true

                        # Patch PVs to remove finalizers
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="dev")].metadata.name}'); do
                            kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                        done

                        # Delete PVs
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="dev")].metadata.name}'); do
                            kubectl delete pv $pv --ignore-not-found || true
                        done

                        # Deploy the Helm charts
                        helm upgrade --install cast-db-dev helm-exam/ --values=helm-exam/values.yaml --namespace dev
                        helm upgrade --install movie-db-dev helm-exam/ --values=helm-exam/values.yaml --namespace dev
                        '''
                    }
                }
            }
        }
        stage('Deployment in QA') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {
                    script {
                        sh '''
                        # Copy kubeconfig file
                        cp $KUBECONFIG /tmp/kubeconfig
                        
                        # Configure the auth of file
                        chmod 600 /tmp/kubeconfig
                        
                        # Configure env
                        export KUBECONFIG=/tmp/kubeconfig
                        
                        # Ensure the namespace is clean
                        kubectl delete namespace qa --ignore-not-found
                        kubectl create namespace qa
                        kubectl config set-context --current --namespace=qa
                        
                        # Delete all resources in the namespace
                        kubectl delete all --all --namespace=qa || true
                        sleep 10
                        
                        # Ensure all PVCs are deleted
                        kubectl delete pvc movie-db-pvc --namespace=qa --ignore-not-found || true
                        kubectl delete pvc cast-db-pvc --namespace=qa --ignore-not-found || true

                        # Patch PVs to remove finalizers
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="qa")].metadata.name}'); do
                            kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                        done

                        # Delete PVs
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="qa")].metadata.name}'); do
                            kubectl delete pv $pv --ignore-not-found || true
                        done
                        
                        # Deploy the Helm charts
                        helm upgrade --install cast-db-qa helm-exam/ --values=helm-exam/values.yaml --namespace qa
                        helm upgrade --install movie-db-qa helm-exam/ --values=helm-exam/values.yaml --namespace qa
                        '''
                    }
                }
            }
        }
        stage('Deployment in staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {
                    script {
                        sh '''
                        # Copy kubeconfig file
                        cp $KUBECONFIG /tmp/kubeconfig
                        
                        # Configure the auth of file
                        chmod 600 /tmp/kubeconfig
                        
                        # Configure env
                        export KUBECONFIG=/tmp/kubeconfig
                        
                        # Ensure the namespace is clean
                        kubectl delete namespace staging --ignore-not-found
                        kubectl create namespace staging
                        kubectl config set-context --current --namespace=staging
                        
                        # Delete all resources in the namespace
                        kubectl delete all --all --namespace=staging || true
                        sleep 10
                        
                        # Ensure all PVCs are deleted
                        kubectl delete pvc movie-db-pvc --namespace=staging --ignore-not-found || true
                        kubectl delete pvc cast-db-pvc --namespace=staging --ignore-not-found || true

                        # Patch PVs to remove finalizers
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="staging")].metadata.name}'); do
                            kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                        done

                        # Delete PVs
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="staging")].metadata.name}'); do
                            kubectl delete pv $pv --ignore-not-found || true
                        done

                        # Deploy the Helm charts
                        helm upgrade --install cast-db-staging helm-exam/ --values=helm-exam/values.yaml --namespace staging
                        helm upgrade --install movie-db-staging helm-exam/ --values=helm-exam/values.yaml --namespace staging
                        '''
                    }
                }
            }
        }
        stage('Deployment in prod') {
            when {
                branch 'master'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you want to deploy to production?', ok: 'Yes'
                }
                withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG')]) {
                    script {
                        sh '''
                        # Copy kubeconfig file
                        cp $KUBECONFIG /tmp/kubeconfig
                        
                        # Configure the auth of file
                        chmod 600 /tmp/kubeconfig
                        
                        # Configure env
                        export KUBECONFIG=/tmp/kubeconfig
                        
                        # Ensure the namespace is clean
                        kubectl delete namespace prod --ignore-not-found
                        kubectl create namespace prod
                        kubectl config set-context --current --namespace=prod
                        
                        # Delete all resources in the namespace
                        kubectl delete all --all --namespace=prod || true
                        sleep 10
                        
                        # Ensure all PVCs are deleted
                        kubectl delete pvc movie-db-pvc --namespace=prod --ignore-not-found || true
                        kubectl delete pvc cast-db-pvc --namespace=prod --ignore-not-found || true

                        # Patch PVs to remove finalizers
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="prod")].metadata.name}'); do
                            kubectl patch pv $pv -p '{"metadata":{"finalizers":null}}' || true
                        done

                        # Delete PVs
                        for pv in $(kubectl get pv -o jsonpath='{.items[?(@.spec.claimRef.namespace=="prod")].metadata.name}'); do
                            kubectl delete pv $pv --ignore-not-found || true
                        done

                        # Deploy the Helm charts
                        helm upgrade --install cast-db-prod helm-exam/ --values=helm-exam/values.yaml --namespace prod
                        helm upgrade --install movie-db-prod helm-exam/ --values=helm-exam/values.yaml --namespace prod
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
            junit 'target/test-*.xml'
        }
        success {
            mail to: 'yumotoayaka@gmail.com',
                 subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' succeeded."
        }
        failure {
            mail to: 'yumotoayaka@gmail.com',
                 subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed."
        }
    }
}
