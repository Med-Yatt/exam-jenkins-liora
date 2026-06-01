pipeline {
    agent any

    environment {
	BUILD_ID = "1"
        DOCKER_ID = "medyatt"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        IMAGE_MOVIE = "${DOCKER_ID}/movie:${DOCKER_TAG}"
        IMAGE_CAST  = "${DOCKER_ID}/cast:${DOCKER_TAG}"
    }

    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                        docker build -t ${IMAGE_MOVIE} ./movie-service
                        docker build -t ${IMAGE_CAST} ./cast-service
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PWD = credentials("DOCKER_HUB_PWD")
            }
            steps {
                script {
                    sh '''
			echo "${DOCKER_PWD}" | docker login -u "${DOCKER_ID}" --password-stdin
                        docker push ${IMAGE_MOVIE}
                        docker push ${IMAGE_CAST}
                    '''
                }
            }
        }

	stage('Déploiement des bases de données') {
            parallel {
                stage('Déploiement Movie DB') {
                    steps {
                        script {
			  sh '''
                            helm upgrade --install movie-db-dev oci://registry-1.docker.io/bitnamicharts/postgresql --namespace dev --create-namespace --set auth.username=movie_db_username --set auth.password=movie_db_password --set auth.database=movie_db_dev
                            helm upgrade --install movie-db-qa oci://registry-1.docker.io/bitnamicharts/postgresql --namespace qa --create-namespace --set auth.username=movie_db_username --set auth.password=movie_db_password --set auth.database=movie_db_qa
                            helm upgrade --install movie-db-staging oci://registry-1.docker.io/bitnamicharts/postgresql --namespace staging --create-namespace --set auth.username=movie_db_username --set auth.password=movie_db_password --set auth.database=movie_db_staging
                            helm upgrade --install movie-db-prod oci://registry-1.docker.io/bitnamicharts/postgresql --namespace prod --create-namespace --set auth.username=movie_db_username --set auth.password=movie_db_password --set auth.database=movie_db_prod
			'''
                        }
                    }
                }
                stage('Déploiement Cast DB') {
                    steps {
                        script {
			  sh '''
                            helm upgrade --install cast-db-dev oci://registry-1.docker.io/bitnamicharts/postgresql --namespace dev --create-namespace --set auth.username=cast_db_username --set auth.password=cast_db_password --set auth.database=cast_db_dev
                            helm upgrade --install cast-db-qa oci://registry-1.docker.io/bitnamicharts/postgresql --namespace qa --create-namespace --set auth.username=cast_db_username --set auth.password=cast_db_password --set auth.database=cast_db_qa
                            helm upgrade --install cast-db-staging oci://registry-1.docker.io/bitnamicharts/postgresql --namespace staging --create-namespace --set auth.username=cast_db_username --set auth.password=cast_db_password --set auth.database=cast_db_staging
                            helm upgrade --install cast-db-prod oci://registry-1.docker.io/bitnamicharts/postgresql --namespace prod --create-namespace --set auth.username=cast_db_username --set auth.password=cast_db_password --set auth.database=cast_db_prod
			'''
                        }
                    }
                }
            }
        }

	stage('Test Databases') {
            steps {
                script {
                    // Vérification de la connexion à movie_db
                    sh '''
                        PGPASSWORD=movie_db_password psql -h movie-db-dev-postgresql -U movie_db_username -d movie_db_dev -c "SELECT 1;"
                        PGPASSWORD=movie_db_password psql -h movie-db-qa-postgresql -U movie_db_username -d movie_db_qa -c "SELECT 1;"
                        PGPASSWORD=movie_db_password psql -h movie-db-staging-postgresql -U movie_db_username -d movie_db_staging -c "SELECT 1;"
                        PGPASSWORD=movie_db_password psql -h movie-db-prod-postgresql -U movie_db_username -d movie_db_prod -c "SELECT 1;"
                    '''

                    // Vérification de la connexion à cast_db
                    sh '''
                        PGPASSWORD=cast_db_password psql -h cast-db-dev-postgresql -U cast_db_username -d cast_db_dev -c "SELECT 1;"
                        PGPASSWORD=cast_db_password psql -h cast-db-qa-postgresql -U cast_db_username -d cast_db_qa -c "SELECT 1;"
                        PGPASSWORD=cast_db_password psql -h cast-db-staging-postgresql -U cast_db_username -d cast_db_staging -c "SELECT 1;"
                        PGPASSWORD=cast_db_password psql -h cast-db-prod-postgresql -U cast_db_username -d cast_db_prod -c "SELECT 1;"
                    '''
                }
            }
        }

        stage('Deploiement sur dev') {
            environment {
                KUBECONFIG = credentials("config-k3s")
            }
            steps {
                script {
		    //sh 'helm upgrade --install projet ./charts --namespace qa --set image.tag=$BUILD_ID'
                    sh '''
                        helm upgrade --install app-dev ./charts --namespace dev --create-namespace \
                          --set movie.image.tag=${DOCKER_TAG} \
                          --set cast.image.tag=${DOCKER_TAG} \
			  --set service.nodePort=30001 \
                    '''
                }
            }
        }

        stage('Deploiement sur QA') {
            environment {
                KUBECONFIG = credentials("config-k3s")
            }
            steps {
                script {
                    sh '''
                        helm upgrade --install app-qa ./charts --namespace qa --create-namespace \
                          --set movie.image.tag=${DOCKER_TAG} \
                          --set cast.image.tag=${DOCKER_TAG} \
			  --set service.nodePort=30002 \
                    '''
                }
            }
        }

        stage('Deploiement sur staging') {
            environment {
                KUBECONFIG = credentials("config-k3s")
            }
            steps {
                script {
                    sh '''
                        helm upgrade --install app-staging ./charts --namespace staging --create-namespace \
                          --set movie.image.tag=${DOCKER_TAG} \
                          --set cast.image.tag=${DOCKER_TAG} \
			  --set service.nodePort=30003 \
                    '''
                }
            }
        }

        stage('Deploiement sur prod') {
            environment {
                KUBECONFIG = credentials("config-k3s")
            }
            when {
		branch 'master'
            }
            steps {
                timeout(time: 15, unit: "MINUTES") {
		    input message: "Déployer en production ?", ok: "Oui"
                }
                script {
                    sh '''
                        helm upgrade --install app-prod ./charts --namespace prod --create-namespace \
                          --set movie.image.tag=${DOCKER_TAG} \
                          --set cast.image.tag=${DOCKER_TAG} \
			  --set service.nodePort=30004 \
                    '''
                }
            }
        }
    }

    post { 
        success {
            echo "Pipeline terminé avec succès"
        }
        failure {
            echo "Envoie de mail en cas d'echec"
            mail to: "amadouyatt@live.fr",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} en echec",
                body: "Pour plus d'infos sur l'echec du pipeline, Merci de verifier la sortie console a ${env.BUILD_URL}"
        }
    }
}
