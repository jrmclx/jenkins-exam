// === FUNCTIONS (usable everywhere in the pipeline) =========================

def buildRunTestPush(imagePrefix, svcDir, port, registry, tag) {
    sh """
    echo "Building ${imagePrefix}..."
    docker build -t ${registry}/${imagePrefix}-api:${tag} ./${svcDir}
    sleep 3

    echo "Creating network ${imagePrefix}-net..."
    docker network rm ${imagePrefix}-net || true
    docker network create ${imagePrefix}-net

    echo "Starting PostgreSQL..."
    docker rm -f ${imagePrefix}-pg || true
    docker run -d --name ${imagePrefix}-pg --network ${imagePrefix}-net \
        -e POSTGRES_USER=db_user \
        -e POSTGRES_PASSWORD=db_pass \
        -e POSTGRES_DB=test-db \
        postgres:12.1-alpine
    sleep 8

    echo "Starting ${imagePrefix}-api..."
    docker rm -f ${imagePrefix}-api || true
    docker run -d -p ${port}:8000 --name ${imagePrefix}-api --network ${imagePrefix}-net \
        -e DATABASE_URI=postgresql://db_user:db_pass@${imagePrefix}-pg/test-db \
        ${registry}/${imagePrefix}-api:${tag}
    sleep 5

    echo "Testing ${imagePrefix}-api..."
    curl -sf -o /dev/null -w "\nHTTP Code : %{http_code}\n" -X GET "http://localhost:${port}/api/v1/${imagePrefix}s/docs"

    echo "Stopping containers..."
    docker stop ${imagePrefix}-api ${imagePrefix}-pg

    echo "Pushing ${imagePrefix}-api..."
    echo $REGISTRY_PASS | docker login ${registry} -u $REGISTRY_USER --password-stdin
    docker tag ${registry}/${imagePrefix}-api:${tag} ${registry}/${imagePrefix}-api:latest
    docker push --all-tags ${registry}/${imagePrefix}-api:${tag}
    """
}

def helmDeploy(release, chart, valuesFile, namespace, extraArgs="") {
    sh """
    helm upgrade --install ${release} ${chart} \
        --values=${valuesFile} \
        --namespace ${namespace} --create-namespace \
        ${extraArgs}
    """
}

def deployEnvironment(namespace, imageTag, nodeport) {
    return {
        stage("Deploy ${namespace}") {
            environment {
                NAMESPACE = namespace
            }
            stages {
                stage('Import Kubeconfig') {
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

                stage('Deploy Movie DB') {
                    when { changeset "**/helm/pgsql/**" } // update statefulsets only if there are changes in PgSQL Helm Chart directory
                    steps {
                        script {
                            helmDeploy(
                                "movie-db", "./helm/pgsql/", "./helm/pgsql/values-movie.yaml", namespace,
                                "--set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PWS"
                            )
                        }
                    }
                }

                stage('Deploy Cast DB') {
                    when { changeset "**/helm/pgsql/**" } // update statefulsets only if there are changes in PgSQL Helm Chart directory
                    steps {
                        script {
                            helmDeploy(
                                "cast-db", "./helm/pgsql/", "./helm/pgsql/values-cast.yaml", namespace,
                                "--set secret.stringData.POSTGRES_USER=$SQL_CREDS_USR --set secret.stringData.POSTGRES_PASSWORD=$SQL_CREDS_PWS"
                            )
                        }
                    }
                }

                stage('Deploy Movie API') {
                    when { changeset "**/movie-service/**" } // update deployment only if there are changes in the movie-service directory
                    steps {
                        script {
                            helmDeploy(
                                "${MOVIE_IMAGE}", "./helm/fastapi/", "./helm/fastapi/values-movie.yaml", namespace,
                                "--set image.tag=${imageTag} --set secret.stringData.DATABASE_URI=postgresql://$SQL_CREDS_USR:$SQL_CREDS_PWS@${MOVIE_SVC_NAME}/movie_db --set secret.stringData.CAST_SERVICE_HOST_URL=http://${CAST_SVC_NAME}/api/v1/casts/"
                            )
                        }
                    }
                }

                stage('Deploy Cast API') {
                    when { changeset "**/cast-service/**" } // update deployment only if there are changes in the cast-service directory
                    steps {
                        script {
                            helmDeploy(
                                "${CAST_IMAGE}", "./helm/fastapi/", "./helm/fastapi/values-cast.yaml", namespace,
                                "--set image.tag=${imageTag} --set secret.stringData.DATABASE_URI=postgresql://$SQL_CREDS_USR:$SQL_CREDS_PWS@${CAST_SVC_NAME}/cast_db"
                            )
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
                        script { // Deploy or update Nginx deployment
                            helmDeploy(
                                "web-frontend", "./helm/nginx/", "./helm/nginx/values.yaml", namespace,
                                "--set service.nodePort=${nodeport}"
                            )
                        }
                    }
                }
            }
        }
    }
}


// === PIPELINE ===============================================================

pipeline {
    agent any

    environment {
        REGISTRY_NAME = "registry.gitlab.com/jrmclx/jenkins-exam"
        REGISTRY_USER = "jrmclx"
        REGISTRY_PASS = credentials("registry-token")
        SQL_CREDS = credentials("pgsql-admin-creds") // this generates also SQL_CREDS_USR & SQL_CREDS_PSW
        KUBECONFIG = credentials("kubeconfig") // we retrieve kubeconfig from Jenkins secret file

        MOVIE_PREFIX   = "movie"
        MOVIE_SVC_DIR = "movie-service"
        MOVIE_SVC_NAME= "${MOVIE_PREFIX}-service"

        CAST_PREFIX   = "cast"
        CAST_SVC_DIR = "cast-service"
        CAST_SVC_NAME= "${CAST_PREFIX}-service"

        IMAGE_TAG = "v${BUILD_ID}.0"
    }

    stages {

        // === Build & Push images in parallel =================================
        stage('Parallel Image Preparation') {
            parallel {
                stage('Movie API') {
                    when { changeset "**/movie-service/**" }
                    steps {
                        script {
                            buildRunTestPush(env.MOVIE_PREFIX, env.MOVIE_SVC_DIR, 8001, env.REGISTRY_NAME, env.IMAGE_TAG)
                        }
                    }
                }
                stage('Cast API') {
                    when { changeset "**/cast-service/**" }
                    steps {
                        script {
                            buildRunTestPush(env.CAST_PREFIX, env.CAST_SVC_DIR, 8002, env.REGISTRY_NAME, env.IMAGE_TAG)
                        }
                    }
                }
            }
        }

        // === Sequential Environments ========================================
        stage('Deployment') {
            stages {
                stage("Non-Prod Environments") {
                    when { expression { env.GIT_BRANCH ==~ /.*(develop|dev|qa|release).*$/ } }
                    stages {
                        
                            deployEnvironment("dev", env.IMAGE_TAG, 30001)()
                            deployEnvironment("qa", env.IMAGE_TAG, 30002)()
                            deployEnvironment("staging", env.IMAGE_TAG, 30003)()
                      
                    }
                }
                 
                stage('Production with Approval') {
                    when { expression { env.GIT_BRANCH ==~ /.*(master|main)$/ } }
                    steps {
                        timeout(time: 15, unit: "MINUTES") { // Button to approve deployment on production (15min timeout)
                            input message: 'Do you want to deploy in production ?', ok: 'Yes'
                        }
                        script {
                            deployEnvironment("prod", env.IMAGE_TAG, 30000)()
                        } 
                    }
                }     
            }
        }
    }
}
