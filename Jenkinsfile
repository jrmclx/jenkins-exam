pipeline {
    environment { // environment variables
        REGISTRY_NAME = "registry.gitlab.com/jrmclx/jenkins-exam"
        REGISTRY_USER = "jrmclx" //clear text for the exercise, but should be stored in Jenkins secret text in a real world scenario
        MOVIE_IMAGE = "movie-api"
        CAST_IMAGE = "cast-api"
        DOCKER_TAG = "v${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    }
    agent any // any available agent
    stages {
        stage('Parallel Image Preparation'){ // Build, Run, Test and Push images in parallel
            environment{
                REGISTRY_PASS = credentials("registry-token") // we retrieve registry token from Jenkins secret text
            }
            parallel {
                stage('Movie image') {
                    when { // run this stage only if there are changes in the movie-service directory
                        changeset "**/movie-service/**"
                    }
                    stages {
                        
                        stage('Building movie-api'){
                            steps{
                                script { //Build Movie API Image
                                    sh '''
                                    echo "Building $MOVIE_IMAGE image..."
                                    docker build -t $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                                    sleep 3
                                    '''
                                }
                            }
                        }
                        
                        stage('Run & Test movie-api'){
                            steps{

                                script { //Docker Network for Movie and PgSQL communication
                                    sh '''
                                    echo "Creating Docker Network movie-net..."
                                    docker network rm movie-net || true
                                    docker network create movie-net
                                    '''
                                }
                                
                                script { //Run Dummy PgSQL
                                    sh '''
                                    echo "Running Postgre image..."
                                    docker rm -f movie-pgsql || true
                                    docker run -d --name movie-pgsql --network movie-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=test-db postgres:12.1-alpine
                                    sleep 8
                                    '''
                                }
                             
                                script { //Run Movie API Image
                                    sh '''
                                    echo "Running $MOVIE_IMAGE..."
                                    docker rm -f $MOVIE_IMAGE || true
                                    docker run -d -p 8001:8000 --name $MOVIE_IMAGE --network movie-net -e DATABASE_URI=postgresql://db_user:db_pass@movie-pgsql/test-db $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG
                                    sleep 5
                                    '''
                                }

                                script { //Test Movie API Image
                                    sh '''
                                    echo "Testing $MOVIE_IMAGE..."
                                    curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:8001/api/v1/movies/docs"
                                    sleep 3
                                    ''' 
                                }

                                script { //Stop containers
                                    sh '''
                                    echo "Stopping containers..."
                                    docker stop $MOVIE_IMAGE
                                    docker stop movie-pgsql
                                    sleep 3
                                    '''
                                }
                            }
                        }
                    
                        stage('Pushing movie-api image'){
                            steps{
                                script { //Push Movie API Image to Registry
                                    sh '''
                                    echo "Pushing $MOVIE_IMAGE image..."
                                    docker tag $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG $REGISTRY_NAME/$MOVIE_IMAGE:latest
                                    echo $REGISTRY_PASS | docker login registry.gitlab.com -u $REGISTRY_USER --password-stdin                                    
                                    docker push $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG
                                    '''
                                }
                            }
                        }
                    }
                }

                stage('Cast image') {
                    when { // run this stage only if there are changes in the cast-service directory
                        changeset "**/cast-service/**"
                    }
                    stages{
                        
                        stage('Building cast-api'){
                            steps{    
                                script { //Build Cast API Image
                                    sh '''
                                    echo "Building $CAST_IMAGE image..."
                                    docker build -t $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG ./cast-service
                                    sleep 3
                                    '''
                                }
                            }
                        }
                        
                        stage('Run & Test cast-api'){
                            steps{

                                script { //Docker Network for Movie and PgSQL communication
                                    sh '''
                                    echo "Creating Docker Network cast-net..."
                                    docker network rm cast-net || true
                                    docker network create cast-net
                                    '''
                                }
                                
                                
                                script { //Run Dummy PgSQL
                                    sh '''
                                    echo "Running Postgre image..."
                                    docker rm -f cast-pgsql || true
                                    docker run -d --name cast-pgsql --network cast-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=test-db postgres:12.1-alpine
                                    sleep 8
                                    '''
                                }
                             
                                script { //Run Movie API Image
                                    sh '''
                                    echo "Running $CAST_IMAGE..."
                                    docker rm -f $CAST_IMAGE || true
                                    docker run -d -p 8002:8000 --name $CAST_IMAGE --network cast-net -e DATABASE_URI=postgresql://db_user:db_pass@cast-pgsql/test-db $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG
                                    sleep 5
                                    '''
                                }

                                script { //Test Movie API Image
                                    sh '''
                                    echo "Testing $CAST_IMAGE..."
                                    curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:8002/api/v1/casts/docs"
                                    sleep 3
                                    ''' 
                                }

                                script { //Stop containers
                                    sh '''
                                    echo "Stopping containers..."
                                    docker stop $CAST_IMAGE
                                    docker stop cast-pgsql
                                    sleep 3
                                    '''
                                }
                            }
                        }
                    
                        stage('Pushing cast-api image'){
                            steps{
                                
                                script { //Push Movie API Image to Registry
                                    sh '''
                                    echo "Pushing ${CAST_IMAGE} image..."
                                    echo $REGISTRY_PASS | docker login registry.gitlab.com -u $REGISTRY_USER --password-stdin
                                    docker tag $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG $REGISTRY_NAME/$CAST_IMAGE:latest
                                    docker push $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG
                                    '''
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Helm Deployment'){
            environment{
                KUBECONFIG = credentials("kubeconfig") // we retrieve kubeconfig from Jenkins secret file
                SQL_CREDS = credentials("pgsql-admin-creds") // this generates also SQL_CREDS_USR & SQL_CREDS_PWS
                
                MOVIE_SVC_NAME = "movie-service"
                CAST_SVC_NAME = "cast-service"

                MOVIE_DB_URI = "postgresql://${SQL_CREDS_USR}:${SQL_CREDS_PWS}@${MOVIE_SVC_NAME}/movie_db"
                CAST_DB_URI = "postgresql://${SQL_CREDS_USR}:${SQL_CREDS_PWS}@${CAST_SVC_NAME}/cast_db"
            }
            stages { // remplace by parallel if you want to deploy envs in parallel
                
                // As we have a single agent for this exercise, we can trigger a unique kubeconfig import for all subsequent stages
                // In a real world scenario with multiple agents, we should import kubeconfig in each stage
                stage('Import Kubeconfig'){ 

                    steps {
                        script { // install or refresh kubeconfig file
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            cat $KUBECONFIG > .kube/config
                            '''
                        }
                    }
                }

                stage('Deploy dev'){
                    environment{
                        NAMESPACE = "dev"
                    }
                    stages {

                        stage('Deploy movie-db'){
                            // when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                            //     changeset "**/helm/pgsql/**"
                            // }
                            steps {
                                steps {
                                    script { // install or refresh kubeconfig file
                                        sh '''
                                        rm -Rf .kube
                                        mkdir .kube
                                        cat $KUBECONFIG > .kube/config
                                        '''
                                    }
                                }                        
                                script { // Deploy movie-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install movie-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-movie.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PWS
                                    '''
                                }
                                script { // Deploy cast-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install cast-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-cast.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PWS
                                    '''
                                }
                            }
                        }

                        stage('Deploy movie-api'){
                            // when { // update deployment only if there are changes in the movie-service directory
                            //     changeset "**/movie-service/**"
                            // }
                            steps {
                                steps {
                                    script { // install or refresh kubeconfig file
                                        sh '''
                                        rm -Rf .kube
                                        mkdir .kube
                                        cat $KUBECONFIG > .kube/config
                                        '''
                                    }
                                } 
                                script { // Deploy movie-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install movie-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-movie.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.DATABASE_URI=$MOVIE_DB_URI \
                                    --set secret.stringData.CAST_SERVICE_HOST_URL="http://$CAST_SVC_NAME/api/v1/casts/"
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-api'){
                            // when { // update deployment only if there are changes in the cast-service directory
                            //     changeset "**/cast-service/**"
                            // }
                            steps {
                                script { // Deploy cast-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install cast-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-cast.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.DATABASE_URI=$CAST_DB_URI
                                    '''
                                }
                            }
                        }
                        
                        stage('Deploy web-frontend'){
                            // when { // update deployment only if there are changes in the Nginx Helm Chart or Nginx conf directory
                            //     anyOf {
                            //         changeset "**/helm/nginx/**"
                            //         changeset "**/nginx/**"
                            //     }
                            // }
                            steps {
                                script { // Update configMaps storing Nginx conf and index files
                                    sh '''
                                    kubectl create configmap nginx-conf --from-file=default.conf=nginx/nginx_config.conf --dry-run=client -o yaml | kubectl apply -f - -n $NAMESPACE
                                    kubectl create configmap nginx-index --from-file=nginx/index.html --dry-run=client -o yaml | kubectl apply -f - -n $NAMESPACE
                                    '''
                                }
                                script { // Deploy web frontend (Nginx) with Helm
                                    sh '''
                                    helm upgrade --install web-frontend ./helm/nginx/ \
                                    --values=./helm/nginx/values.yaml \
                                    --namespace $NAMESPACE --create-namespace
                                    '''
                                }
                            }
                        }
                    }
                }
            }
        // stage('Deploiement en prod'){
        //     environment{
        //         KUBECONFIG = credentials("kubeconfig") // we retrieve kubeconfig from Jenkins secret file
        //     }
        //     steps {
        //         timeout(time: 15, unit: "MINUTES") { // Create an Approval Button with a timeout of 15minutes to approve deployment on production
        //             input message: 'Do you want to deploy in production ?', ok: 'Yes'
        //         }

        //         script {
        //         sh '''
        //             rm -Rf .kube
        //             mkdir .kube
        //             ls
        //             cat $KUBECONFIG > .kube/config
        //             cp fastapi/values.yaml values.yml
        //             cat values.yml
        //             sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
        //             helm upgrade --install app fastapi --values=values.yml --namespace prod
        //             '''
        //         }
        //     }

        }
    }
}
