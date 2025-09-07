pipeline {
    environment { // environment variables
        REGISTRY_NAME = "registry.gitlab.com/jrmclx/jenkins-exam"
        REGISTRY_USER = "jrmclx" //clear text for the exercise, but should be stored in Jenkins secret text in a real world scenario

        MOVIE_PREFIX = "movie"
        CAST_PREFIX = "cast"

        MOVIE_IMAGE = "${MOVIE_PREFIX}-api"
        MOVIE_SVC_NAME = "${MOVIE_PREFIX}-service"

        CAST_IMAGE = "${CAST_PREFIX}-api"
        CAST_SVC_NAME = "${CAST_PREFIX}-service"

        IMAGE_TAG = "v${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    }

    agent any // any available agent

    stages {


        stage('Debug Branch') {
            steps {
                echo "BRANCH_NAME = ${env.BRANCH_NAME}"
                echo "GIT_BRANCH  = ${env.GIT_BRANCH}"
            }
        }

        stage('Parallel Image Preparation'){ // Build, Run, Test and Push images in parallel
            environment{
                REGISTRY_PASS = credentials("registry-token") // we retrieve registry token from Jenkins secret text
            }
            parallel {
                stage('Movie image') {
                    when { // run this stage only if there are changes in the movie-service directory
                        changeset "**/movie-service/**"
                    }
                    environment{
                        IMAGE_NAME = "$MOVIE_IMAGE"
                        SVC_NAME = "$MOVIE_SVC_NAME"
                        PREFIX = "$MOVIE_PREFIX"
                    }
                    stages {
                        
                        stage('Build movie-api'){
                            steps{
                                script { //Build Movie API Image
                                    sh '''
                                    echo "Building $IMAGE_NAME image..."
                                    docker build -t $REGISTRY_NAME/$IMAGE_NAME:$IMAGE_TAG ./$SVC_NAME
                                    sleep 3
                                    '''
                                }
                            }
                        }
                        
                        stage('Run/Test movie-api'){
                            steps{

                                script { //Docker Network for Movie and PgSQL communication
                                    sh '''
                                    echo "Creating Docker Network ${PREFIX}-net..."
                                    docker network rm ${PREFIX}-net || true
                                    docker network create ${PREFIX}-net
                                    '''
                                }
                                
                                script { //Run Dummy PgSQL
                                    sh '''
                                    echo "Running Postgre image..."
                                    docker rm -f ${PREFIX}-pgsql || true
                                    docker run -d --name ${PREFIX}-pgsql --network ${PREFIX}-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=test-db postgres:12.1-alpine
                                    sleep 8
                                    '''
                                }
                             
                                script { //Run Movie API Image
                                    sh '''
                                    echo "Running $IMAGE_NAME..."
                                    docker rm -f $IMAGE_NAME || true
                                    docker run -d -p 8001:8000 --name $IMAGE_NAME --network ${PREFIX}-net -e DATABASE_URI=postgresql://db_user:db_pass@${PREFIX}-pgsql/test-db $REGISTRY_NAME/$IMAGE_NAME:$IMAGE_TAG
                                    sleep 5
                                    '''
                                }

                                script { //Test Movie API Image
                                    sh '''
                                    echo "Testing $IMAGE_NAME..."
                                    curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:8001/api/v1/${PREFIX}s/docs"
                                    sleep 3
                                    ''' 
                                }

                                script { //Stop containers
                                    sh '''
                                    echo "Stopping containers..."
                                    docker stop $IMAGE_NAME
                                    docker stop ${PREFIX}-pgsql
                                    sleep 3
                                    '''
                                }
                            }
                        }
                    
                        stage('Pushing movie-api image'){
                            steps{
                                script { //Push Movie API Image to Registry
                                    sh '''
                                    echo "Pushing $IMAGE_NAME image..."
                                    docker tag $REGISTRY_NAME/$IMAGE_NAME:$IMAGE_TAG $REGISTRY_NAME/$IMAGE_NAME:latest
                                    echo $REGISTRY_PASS | docker login registry.gitlab.com -u $REGISTRY_USER --password-stdin                                    
                                    docker push $REGISTRY_NAME/$IMAGE_NAME:$IMAGE_TAG
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
                    environment{
                        IMAGE_NAME = "$CAST_IMAGE"
                        SVC_NAME = "$CAST_SVC_NAME"
                        PREFIX = "$CAST_PREFIX"
                    }
                    stages{
                        
                        stage('Building cast-api'){
                            steps{    
                                script { //Build Cast API Image
                                    sh '''
                                    echo "Building $IMAGE_NAME image..."
                                    docker build -t $REGISTRY_NAME/$IMAGE_NAME:$IMAGE_TAG ./$SVC_NAME
                                    sleep 3
                                    '''
                                }
                            }
                        }
                        
                        stage('Run & Test cast-api'){
                            steps{

                                script { //Docker Network for Movie and PgSQL communication
                                    sh '''
                                    echo "Creating Docker Network ${PREFIX}-net..."
                                    docker network rm ${PREFIX}-net || true
                                    docker network create ${PREFIX}-net
                                    '''
                                }
                                
                                
                                script { //Run Dummy PgSQL
                                    sh '''
                                    echo "Running Postgre image..."
                                    docker rm -f ${PREFIX}-pgsql || true
                                    docker run -d --name ${PREFIX}-pgsql --network ${PREFIX}-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=test-db postgres:12.1-alpine
                                    sleep 8
                                    '''
                                }
                             
                                script { //Run Movie API Image
                                    sh '''
                                    echo "Running $IMAGE_NAME..."
                                    docker rm -f $IMAGE_NAME || true
                                    docker run -d -p 8002:8000 --name $IMAGE_NAME --network ${PREFIX}-net -e DATABASE_URI=postgresql://db_user:db_pass@${PREFIX}-pgsql/test-db $REGISTRY_NAME/$IMAGE_NAME:$IMAGE_TAG
                                    sleep 5
                                    '''
                                }

                                script { //Test Movie API Image
                                    sh '''
                                    echo "Testing $IMAGE_NAME..."
                                    curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:8002/api/v1/${PREFIX}s/docs"
                                    sleep 3
                                    ''' 
                                }

                                script { //Stop containers
                                    sh '''
                                    echo "Stopping containers..."
                                    docker stop $IMAGE_NAME
                                    docker stop ${PREFIX}-pgsql
                                    sleep 3
                                    '''
                                }
                            }
                        }
                    
                        stage('Pushing cast-api image'){
                            steps{
                                script { //Push Movie API Image to Registry
                                    sh '''
                                    echo "Pushing $IMAGE_NAME image..."
                                    echo $REGISTRY_PASS | docker login registry.gitlab.com -u $REGISTRY_USER --password-stdin
                                    docker tag $REGISTRY_NAME/$CAST_IMAGE:$IMAGE_TAG $REGISTRY_NAME/$IMAGE_NAME:latest
                                    docker push $REGISTRY_NAME/$CAST_IMAGE:$IMAGE_TAG
                                    '''
                                }
                            }
                        }
                    }
                }
            }
        }

        // DEPLOYMENT STAGES ------------------------------------------------------------------------
        stage('Deployment'){
            environment{
                KUBECONFIG = credentials("kubeconfig") // we retrieve kubeconfig from Jenkins secret file
                SQL_CREDS = credentials("pgsql-admin-creds") // this generates also SQL_CREDS_USR & SQL_CREDS_PSW
                
                MOVIE_DB_SVC = "${MOVIE_PREFIX}-db"
                CAST_DB_SVC = "${CAST_PREFIX}-db"

                MOVIE_DB_URI = "postgresql://${SQL_CREDS_USR}:${SQL_CREDS_PSW}@${MOVIE_DB_SVC}/${MOVIE_PREFIX}_db"
                CAST_DB_URI = "postgresql://${SQL_CREDS_USR}:${SQL_CREDS_PSW}@${CAST_DB_SVC}/${CAST_PREFIX}_db"

                NODEPORT_DEV = 30001
                NODEPORT_QA = 30002
                NODEPORT_STG = 30003
                NODEPORT_PROD = 30000
            }

            when { // update deployment only if there are changes in the Helm Charts or App source code
                anyOf {
                    changeset "**/nginx/**"
                    changeset "**/movie-service/**"
                    changeset "**/cast-service/**"
                    changeset "**/helm/**"
                }
            }

            parallel { // replace by 'stages' if you want a sequential deployment
                
                stage('Deploy dev'){
                    environment{
                        NAMESPACE = "dev"
                        NODEPORT = "$NODEPORT_DEV"
                    }

                    stages {

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

                        stage('Deploy movie-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                            }
                            steps {
                     
                                script { // Deploy movie-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                            }
                            steps {
                                script { // Deploy cast-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy movie-api'){
                            when { // update deployment only if there are changes in the movie-service directory
                                changeset "**/movie-service/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                                DB_URI = "$MOVIE_DB_URI"
                            }
                            steps {

                                script { // Deploy movie-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI \
                                    --set secret.stringData.CAST_SERVICE_HOST_URL="http://$CAST_SVC_NAME/api/v1/casts/"
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-api'){
                            when { // update deployment only if there are changes in the cast-service directory
                                changeset "**/cast-service/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                                DB_URI = "$CAST_DB_URI"
                            }
                            steps {
                                script { // Deploy cast-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI
                                    '''
                                }
                            }
                        }
                        
                        stage('Deploy web-frontend'){
                            when { // update deployment only if there are changes in the Nginx Helm Chart or Nginx conf directory
                                anyOf {
                                    changeset "**/helm/nginx/**"
                                    changeset "**/nginx/**"
                                }
                            }
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
                                    --namespace $NAMESPACE --create-namespace \
                                    --set service.nodePort=$NODEPORT
                                    '''
                                }
                            }
                        }
                    }
                }

                stage('Deploy qa'){
                    environment{
                        NAMESPACE = "qa"
                        NODEPORT = "$NODEPORT_QA"
                    }
                    
                    stages {

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

                        stage('Deploy movie-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                            }
                            steps {
                     
                                script { // Deploy movie-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                            }
                            steps {
                                script { // Deploy cast-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy movie-api'){
                            when { // update deployment only if there are changes in the movie-service directory
                                changeset "**/movie-service/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                                DB_URI = "$MOVIE_DB_URI"
                            }
                            steps {

                                script { // Deploy movie-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI \
                                    --set secret.stringData.CAST_SERVICE_HOST_URL="http://$CAST_SVC_NAME/api/v1/casts/"
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-api'){
                            when { // update deployment only if there are changes in the cast-service directory
                                changeset "**/cast-service/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                                DB_URI = "$CAST_DB_URI"
                            }
                            steps {
                                script { // Deploy cast-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI
                                    '''
                                }
                            }
                        }
                        
                        stage('Deploy web-frontend'){
                            when { // update deployment only if there are changes in the Nginx Helm Chart or Nginx conf directory
                                anyOf {
                                    changeset "**/helm/nginx/**"
                                    changeset "**/nginx/**"
                                }
                            }
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
                                    --namespace $NAMESPACE --create-namespace \
                                    --set service.nodePort=$NODEPORT
                                    '''
                                }
                            }
                        }
                    }
                }

                stage('Deploy stg'){
                    environment{
                        NAMESPACE = "staging"
                        NODEPORT = "$NODEPORT_STG"
                    }

                    stages {

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

                        stage('Deploy movie-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                            }
                            steps {
                     
                                script { // Deploy movie-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                            }
                            steps {
                                script { // Deploy cast-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy movie-api'){
                            when { // update deployment only if there are changes in the movie-service directory
                                changeset "**/movie-service/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                                DB_URI = "$MOVIE_DB_URI"
                            }
                            steps {

                                script { // Deploy movie-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI \
                                    --set secret.stringData.CAST_SERVICE_HOST_URL="http://$CAST_SVC_NAME/api/v1/casts/"
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-api'){
                            when { // update deployment only if there are changes in the cast-service directory
                                changeset "**/cast-service/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                                DB_URI = "$CAST_DB_URI"
                            }
                            steps {
                                script { // Deploy cast-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI
                                    '''
                                }
                            }
                        }
                        
                        stage('Deploy web-frontend'){
                            when { // update deployment only if there are changes in the Nginx Helm Chart or Nginx conf directory
                                anyOf {
                                    changeset "**/helm/nginx/**"
                                    changeset "**/nginx/**"
                                }
                            }
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
                                    --namespace $NAMESPACE --create-namespace \
                                    --set service.nodePort=$NODEPORT
                                    '''
                                }
                            }
                        }
                    }
                }

                stage('Deploy prod'){
                    environment{
                        NAMESPACE = "prod"
                        NODEPORT = "$NODEPORT_PROD"
                    }
                    // when { // deploy to production only if there are changes in Master branch
                    //     branch pattern: ".*(master|main)", comparator: "REGEXP"
                    // }

                    stages {

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

                        stage('Approval to deploy prod'){
                            steps {
                                timeout(time: 15, unit: "MINUTES") { // Button to approve deployment on production (15min timeout)
                                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                                }
                            }
                        }
                        
                        stage('Deploy movie-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                            }
                            steps {
                     
                                script { // Deploy movie-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }
                        stage('Deploy cast-db'){
                            when { // update statefulsets only if there are changes in PgSQL Helm Chart directory
                                changeset "**/helm/pgsql/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                            }
                            steps {
                                script { // Deploy cast-db (PostgreSQL) with Helm --- Override user and password values from Jenkins secrets
                                    sh '''
                                    helm upgrade --install ${PREFIX}-db ./helm/pgsql/ \
                                    --values=./helm/pgsql/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR \
                                    --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PSW
                                    '''
                                }
                            }
                        }

                        stage('Deploy movie-api'){
                            when { // update deployment only if there are changes in the movie-service directory
                                changeset "**/movie-service/**"
                            }
                            environment{
                                PREFIX = "$MOVIE_PREFIX"
                                DB_URI = "$MOVIE_DB_URI"
                            }
                            steps {

                                script { // Deploy movie-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI \
                                    --set secret.stringData.CAST_SERVICE_HOST_URL="http://$CAST_SVC_NAME/api/v1/casts/"
                                    '''
                                }
                            }
                        }

                        stage('Deploy cast-api'){
                            when { // update deployment only if there are changes in the cast-service directory
                                changeset "**/cast-service/**"
                            }
                            environment{
                                PREFIX = "$CAST_PREFIX"
                                DB_URI = "$CAST_DB_URI"
                            }
                            steps {
                                script { // Deploy cast-api (FastAPI) with Helm
                                    sh '''
                                    helm upgrade --install ${PREFIX}-api ./helm/fastapi/ \
                                    --values=./helm/fastapi/values-${PREFIX}.yaml \
                                    --namespace $NAMESPACE --create-namespace \
                                    --set image.tag="$IMAGE_TAG" \
                                    --set secret.stringData.DATABASE_URI=$DB_URI
                                    '''
                                }
                            }
                        }
                        
                        stage('Deploy web-frontend'){
                            when { // update deployment only if there are changes in the Nginx Helm Chart or Nginx conf directory
                                anyOf {
                                    changeset "**/helm/nginx/**"
                                    changeset "**/nginx/**"
                                }
                            }
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
                                    --namespace $NAMESPACE --create-namespace \
                                    --set service.nodePort=$NODEPORT
                                    '''
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
