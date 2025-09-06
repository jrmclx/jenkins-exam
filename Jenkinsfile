pipeline {
    environment { // environment variables
        REGISTRY_NAME = "registry.gitlab.com/jrmclx/jenkins-exam"
        REGISTRY_USER = "jrmclx"
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
                stage('MOVIE Image Prep') {
                    stages {
                        
                        stage('Building ${MOVIE_IMAGE} image'){
                            steps{
                                echo "Building ${MOVIE_IMAGE} image..."
                                script { //Build Movie API Image
                                    sh '''
                                    docker build -t $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                                    sleep 3
                                    '''
                                }
                            }
                        }
                        
                        stage('Running ${MOVIE_IMAGE} image'){
                            steps{

                                echo "Creating Docker Network movie-net..."
                                script { //Docker Network for Movie and PgSQL communication
                                    sh '''
                                    docker network rm movie-net || true
                                    docker network create movie-net
                                    '''
                                }

                                echo "Running Postgre image..."
                                script { //Run Dummy PgSQL
                                    sh '''
                                    docker rm -f movie-pgsql || true
                                    docker run -d --name movie-pgsql --network test-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=test-db postgres:12.1-alpine
                                    sleep 8
                                    '''
                                }

                                echo "Running ${MOVIE_IMAGE} image..."
                                script { //Run Movie API Image
                                    sh '''
                                    docker rm -f $MOVIE_IMAGE || true
                                    docker run -d -p 8001:8000 --name $MOVIE_IMAGE --network movie-net -e DATABASE_URI=postgresql://db_user:db_pass@movie-pgsql/test-db $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG
                                    sleep 3
                                    '''
                                }
                            }
                        }

                        stage('Testing ${MOVIE_IMAGE} image'){
                            steps{
                                echo "Testing ${MOVIE_IMAGE} image..."
                                script { //Test Movie API Image
                                    sh '''
                                    curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:8001/api/v1/movies/docs"
                                    sleep 3
                                    ''' 
                                }
                            }
                        }

                        stage('Stopping ${MOVIE_IMAGE} image'){
                            steps{
                                echo "Stopping ${MOVIE_IMAGE} image..."
                                script { //Stop containers
                                    sh '''
                                    docker stop $MOVIE_IMAGE
                                    docker stop movie-pgsql
                                    sleep 3
                                    docker network rm movie-net
                                    '''
                                }
                            }
                        }
                    
                        stage('Pushing ${MOVIE_IMAGE} image'){
                            steps{
                                echo "Pushing ${MOVIE_IMAGE} image..."
                                script { //Push Movie API Image to Registry
                                    sh '''
                                    echo $REGISTRY_PASS | docker login registry.gitlab.com -u $REGISTRY_USER --password-stdin
                                    docker tag $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG $REGISTRY_NAME/$MOVIE_IMAGE:latest
                                    docker push $REGISTRY_NAME/$MOVIE_IMAGE:$DOCKER_TAG
                                    '''
                                }
                            }
                        }
                    }
                }
                stage('CAST Image Prep') {
                    steps {
                        echo "Building ${CAST_IMAGE} image..."
                        script { //Build Cast API Image
                            sh '''
                            docker build -t $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG ./cast-service
                            sleep 3
                            '''
                        }
                        echo "Creating Docker Network cast-net..."
                        script { //Docker Network for Cast and PgSQL communication
                            sh '''
                            docker network rm cast-net || true
                            docker network create cast-net
                            '''
                        }
                        echo "Running Postgre image..."
                        script { //Run Dummy PgSQL
                            sh '''
                            docker rm -f cast-pgsql || true
                            docker run -d --name cast-pgsql --network cast-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=test-db postgres:12.1-alpine
                            sleep 8
                            '''
                        }
                        echo "Running ${CAST_IMAGE} image..."
                        script { //Run Cast API Image
                            sh '''
                            docker rm -f $CAST_IMAGE || true
                            docker run -d -p 8002:8000 --name $CAST_IMAGE --network cast-net -e DATABASE_URI=postgresql://db_user:db_pass@cast-pgsql/test-db $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG
                            sleep 8
                            '''
                        }
                        echo "Testing ${CAST_IMAGE} image..."
                        script { //Test Cast API Image
                            sh '''
                            curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:8002/api/v1/casts/docs"
                            sleep 3
                            ''' 
                        }
                        echo "Stopping ${CAST_IMAGE} image..."
                        script { //Test Cast API Image
                            sh '''
                            docker stop $CAST_IMAGE
                            sleep 3
                            ''' 
                        }
                        echo "Pushing ${CAST_IMAGE} image..."
                        script { //Push Cast API Image to Registry
                            sh '''
                            echo $REGISTRY_PASS | docker login registry.gitlab.com -u $REGISTRY_USER --password-stdin
                            docker tag $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG $REGISTRY_NAME/$CAST_IMAGE:latest
                            docker push $REGISTRY_NAME/$CAST_IMAGE:$DOCKER_TAG
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploiement en dev'){
            environment{
                KUBECONFIG = credentials("kubeconfig") // we retrieve kubeconfig from Jenkins secret file
            }
            steps {
                script { // install or refresh kubeconfig file
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    '''
                }
                script { // Deploy movie-db (PostgreSQL) with Helm
                    sh 'helm upgrade --install movie-db ./helm/pgsql/ --values=./helm/pgsql/values-movie.yaml --namespace dev'
                }
                script { // Deploy cast-db (PostgreSQL) with Helm
                    sh 'helm upgrade --install cast-db ./helm/pgsql/ --values=./helm/pgsql/values-cast.yaml --namespace dev'
                }
                script { // Deploy movie-api (FastAPI) with Helm
                    sh 'helm upgrade --install movie-api ./helm/fastapi/ --values=./helm/pgsql/values-movie.yaml --namespace dev'
                }
                script { // Deploy cast-api (FastAPI) with Helm
                    sh 'helm upgrade --install cast-api ./helm/fastapi/ --values=./helm/pgsql/values-cast.yaml --namespace dev'
                }
                script { // Deploy web frontend (Nginx) with Helm
                    sh 'helm upgrade --install web-frontend ./helm/nginx/ --values=./helm/nginx/values.yaml --namespace dev'
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
