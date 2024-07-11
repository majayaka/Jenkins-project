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
        stage('Docker Build') { 
            steps {
                script {
                sh '''
                 docker rm -f jenkins-cast jenkins-movie || true
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
                 sleep 6
                '''
                }
            }
        }
        stage('Docker Run Images') {
            steps {
                script {
                sh '''
                    docker run -d -p 8002:8000 --name jenkins-cast $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                    docker run -d -p 8001:8000 --name jenkins-movie $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                    sleep 10
                '''
                }
            }
        }

        stage('Docker Test Containers') {
            steps {
                script {
                    sh '''
                     # Test if cast-service is running
                     if ! docker ps -a | grep -q jenkins-cast; then
                        echo "Cast service container failed to start."
                        exit 1
                     fi

                     # Test if movie-service is running
                     if ! docker ps -a | grep -q jenkins-movie; then
                        echo "Movie service container failed to start."
                        exit 1
                     fi

                     echo "Both containers are running successfully."
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
                     docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:latest
                     docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                     docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:latest
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

                            # Fix PVC metadata if it exists
                            if kubectl get pvc movie-db-pvc --namespace=dev; then
                                kubectl annotate pvc movie-db-pvc meta.helm.sh/release-name=movie-db-dev --overwrite
                                kubectl annotate pvc movie-db-pvc meta.helm.sh/release-namespace=dev --overwrite
                            fi

                            # Deploy the Helm charts
                            helm upgrade --install cast-db-dev helm-exam/ --values=helm-exam/values.yaml --namespace dev
                            helm upgrade --install movie-db-dev helm-exam/ --values=helm-exam/values.yaml --namespace dev
                        '''
                    }
                }
            }
        }
        // Similar stages for QA, staging, and prod with appropriate namespace changes
    }
    post {
        always {
            archiveArtifacts artifacts: 'target/test-*.xml', allowEmptyArchive: true
            junit 'target/test-*.xml'
        }
        failure {
            emailext to: 'your-email@example.com',
                 subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' failed."
        }
    }
}
