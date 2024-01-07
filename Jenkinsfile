pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID="wldhub71" // docker-id for dockerhub repository for images
        DOCKER_CAST_IMAGE="cast-service"
        DOCKER_MOVIE_IMAGE="movie-service"
        DOCKER_TAG = "v.${BUILD_ID}.0" // image with the current build - increment by 1 
    }
    agent any // Jenkins will select all available agents
    stages {
        stage('Docker Build'){ // build image stage
            steps {
                script {
                    sh '''
                    docker rm -f cast-service  movie-service cast-db movie-db nginx
                    docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG ./cast-service
                    docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                    '''
                }
            }
        }
    
        stage('Docker Run'){ // run container with new image
            steps {
                script {
                    sh '''
                    docker run -d --name movie-db -v postgres_data_movie:/var/lib/postgresql/data/ -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev -d postgres:12.1-alpine
                    docker run -d --name cast-db -v postgres_data_cast:/var/lib/postgresql/data/ -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev -d postgres:12.1-alpine
                    docker run -d --name movie-service -p 8001:8000 -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-db/movie_db_dev -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ --link movie-db $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    docker run -d --name cast-service -p 8002:8000 -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-db/cast_db_dev --link cast-db $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                    docker run -d --name nginx -p 8081:8080 -v /$(pwd)/nginx_config.conf:/etc/nginx/conf.d/default.conf --link cast-service --link movie-service -d nginx:latest
                     '''
                }
            }
        }

        stage('Test Acceptance'){ // Test new images by using NGINX proxy to access application  
            steps {
                script {
                    sh '''
                    curl localhost:8081
                    '''
                }
            }
        }
    
        stage('Docker Push'){ // Push new validated images to dockerhub repository
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") //  docker credentials from docker_hub_pass variable in Jenkins
            }
            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                     '''
                }
            }
        }

        stage('Deploy To DEV'){
            environment {
                KUBECONFIG = credentials("config") // kubeconfig from config variable in Jenkins
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config

                    helm -n dev upgrade --install movie-db --values helm-db/values-movie.yaml helm-db/
                    helm -n dev upgrade --install cast-db --values helm-db/values-cast.yaml helm-db/
                    sleep 5
                    helm -n dev upgrade --install movie-service --values helm-movie-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set app_image.tag=$DOCKER_TAG helm-movie-service/
                    helm -n dev upgrade --install cast-service --values helm-cast-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set app_image.tag=$DOCKER_TAG helm-cast-service/
                    helm -n dev upgrade --install nginx --values helm-nginx/values.yaml --set nginx.nodeport.nodeport=30180 helm-nginx/
                    '''
                }
            }
        }

        stage('Deploy To QA'){
            environment {
                KUBECONFIG = credentials("config") // kubeconfig from config variable in Jenkins
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config

                    helm -n qa upgrade --install movie-db --values helm-db/values-movie.yaml helm-db/
                    helm -n qa upgrade --install cast-db --values helm-db/values-cast.yaml helm-db/
                    sleep 6
                    helm -n qa upgrade --install movie-service --values helm-movie-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set app_image.tag=$DOCKER_TAG helm-movie-service/
                    helm -n qa upgrade --install cast-service --values helm-cast-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set app_image.tag=$DOCKER_TAG helm-cast-service/
                    helm -n qa upgrade --install nginx --values helm-nginx/values.yaml --set nginx.nodeport.nodeport=30280 helm-nginx/

                    '''
                }
            }
        }

        stage('Deploy To STAGING'){
            environment {
                KUBECONFIG = credentials("config") // kubeconfig from config variable in Jenkins
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config

                    helm -n staging upgrade --install movie-db --values helm-db/values-movie.yaml helm-db/
                    helm -n staging upgrade --install cast-db --values helm-db/values-cast.yaml helm-db/
                    sleep 5
                    helm -n staging upgrade --install movie-service --values helm-movie-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set app_image.tag=$DOCKER_TAG helm-movie-service/
                    helm -n staging upgrade --install cast-service --values helm-cast-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set app_image.tag=$DOCKER_TAG helm-cast-service/
                    helm -n staging upgrade --install nginx --values helm-nginx/values.yaml --set nginx.nodeport.nodeport=30380 helm-nginx/

                    '''
                }
            }
        }
  
        stage('Deploy To PROD'){
            environment {
                KUBECONFIG = credentials("config") // kubeconfig from config variable in Jenkins
            }
            when {
                expression {
                    return env.GIT_BRANCH == 'origin/main';
                }
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
                        cat $KUBECONFIG > .kube/config
                        helm -n prod upgrade --install movie-db --values helm-db/values-movie.yaml helm-db/
                        helm -n prod upgrade --install cast-db --values helm-db/values-cast.yaml helm-db/
                        sleep 10
                        helm -n prod upgrade --install movie-service --values helm-movie-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set app_image.tag=$DOCKER_TAG helm-movie-service/
                        helm -n prod upgrade --install cast-service --values helm-cast-service/values.yaml --set app_image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set app_image.tag=$DOCKER_TAG helm-cast-service/
                        helm -n prod upgrade --install nginx --values helm-nginx/values.yaml --set nginx.nodeport.nodeport=30480 helm-nginx/
                    '''
                }
            }
        }
    }
}
