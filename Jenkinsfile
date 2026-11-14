// Pipeline des TP 3.5 et 3.6 (Baptiste Kaldjob) : tests -> build -> test d'acceptance -> push (main uniquement)
pipeline {
    agent any

    parameters {
        string(name: 'DOCKERHUB_ID', defaultValue: 'kaldjobbaptiste', description: 'Votre identifiant Docker Hub')
        booleanParam(name: 'PUSH', defaultValue: true, description: "Pousser l'image sur Docker Hub")
        choice(name: 'APP_COLOR', choices: ['#1f6feb', '#2da44e', '#d1242f'], description: 'Couleur de la bannière (défi TP 3.5)')
    }

    environment {
        IMAGE          = "${params.DOCKERHUB_ID}/legeni-webapp"
        // TP 3.6 : main -> 1.1.<n> (1.0.x = builds du TP 3.5), autre branche -> <branche>-<n>
        VERSION        = "${(env.BRANCH_NAME ?: 'main') == 'main' ? '1.1.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"
        // nom de branche dans les noms : main #3 et PR-1 #3 peuvent tourner en même temps
        TEST_CONTAINER = "tp3.6-webapp-ci-${(env.BRANCH_NAME ?: 'main').replaceAll('[^a-zA-Z0-9.-]', '-')}-${env.BUILD_NUMBER}"
        CI_NETWORK     = "tp3.6-ci-net-${(env.BRANCH_NAME ?: 'main').replaceAll('[^a-zA-Z0-9.-]', '-')}-${env.BUILD_NUMBER}"
        // params.* a toujours sa valeur par défaut, même pour un build lancé par le Poll SCM
        COULEUR        = "${params.APP_COLOR}"
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Tests unitaires') {
            steps {
                // L'étape "test" du Dockerfile lance flake8 + pytest : si un test échoue, le build s'arrête ici
                sh 'docker build --target test -t legeni-webapp:test .'
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION -t $IMAGE:latest .'
            }
        }

        stage("Test d'acceptance") {
            steps {
                sh '''
                    docker network create $CI_NETWORK
                    docker run -d --name $TEST_CONTAINER --network $CI_NETWORK -e APP_COLOR="$COULEUR" $IMAGE:$VERSION
                    # On interroge le conteneur depuis un conteneur curl placé sur le même réseau
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf --retry 10 --retry-connrefused --retry-delay 1 http://$TEST_CONTAINER:5000/health
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf http://$TEST_CONTAINER:5000/ | grep -q "Version $VERSION"
                    # Défi : la couleur choisie apparaît bien dans la page
                    docker run --rm --network $CI_NETWORK curlimages/curl:8.11.1 \
                        -sf http://$TEST_CONTAINER:5000/ | grep -q "$COULEUR" && echo "couleur $COULEUR présente dans la page"
                '''
            }
        }

        stage('Push Docker Hub') {
            when { allOf { branch 'main'; expression { params.PUSH } } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$VERSION
                        docker push $IMAGE:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f $TEST_CONTAINER || true; docker network rm $CI_NETWORK || true'
        }
        success {
            echo "Image ${IMAGE}:${VERSION} construite et validée"
        }
        failure {
            echo "Échec du pipeline : consultez la console de l'étape en rouge"
        }
    }
}
