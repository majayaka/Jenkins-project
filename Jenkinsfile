pipeline {
    environment {
        DOCKER_ID = "ayakayu"
        DOCKER_CAST_IMAGE = "cast-service"
        DOCKER_MOVIE_IMAGE = "movie-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    agent any
    stages {
        stage('Docker Build Cast Image') {
            steps {
                script {
                    sh '''
                      docker rm -f jenkins || true
                      docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG cast-service/
                      sleep 6
                    '''
                }
            }
        }

        stage('Docker Build Movie Image') {
            steps {
                script {
                    sh '''
                      docker rm -f jenkins || true
                      docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG movie-service/
                      sleep 6
                    '''
                }
            }
        }

        stage('Docker Run Cast Image') {
            steps {
                script {
                    sh '''
                     docker run -d -p 8002:8000 --name jenkins-cast $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                     sleep 10
                    '''
                }
            }
        }

        stage('Docker Run Movie Image') {
            steps {
                script {
                    sh '''
                     docker run -d -p 8001:8000 --name jenkins-movie $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                     sleep 10
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                     docker login -u $DOCKER_ID -p $DOCKER_PASS
                     docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                     docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }
    }
}