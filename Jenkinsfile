// TP 3.8 (Baptiste Kaldjob) : intégration continue + déploiement continu (staging automatique, prod après validation)
pipeline {
    agent any

    environment {
        IMAGE   = "legeni-webapp"
        // main -> 1.2.<n> ; autre branche ou PR -> <branche>-<n> (tags locaux uniques, utiles au rollback)
        VERSION = "${(env.BRANCH_NAME ?: 'main') == 'main' ? '1.2.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"
        COMPOSE = "docker compose -f deploy/docker-compose.deploy.yml"
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()          // jamais deux déploiements en même temps
    }

    stages {
        stage('Tests unitaires') {
            steps {
                sh 'docker build --target test -t $IMAGE:test .'
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$VERSION -t $IMAGE:$VERSION .'
            }
        }

        stage('Déploiement staging') {
            environment {
                REDIS_PASSWORD = credentials('redis-password-staging')
            }
            steps {
                sh '''
                    IMAGE_REF=$IMAGE:$VERSION APP_ENV=staging APP_COLOR='#bf8700' PORT=8081 \
                        $COMPOSE -p tp3-8-staging up -d --wait
                '''
                sh './deploy/smoke-test.sh tp3-8-staging $VERSION'
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Déployer la version ${VERSION} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            environment {
                REDIS_PASSWORD = credentials('redis-password-prod')
            }
            steps {
                script {
                    // Image actuellement en production : servira au rollback
                    env.PREVIOUS_IMAGE = sh(
                        script: "docker inspect -f '{{.Config.Image}}' tp3-8-prod-web-1 2>/dev/null || echo none",
                        returnStdout: true
                    ).trim()
                    echo "Version en production avant déploiement : ${env.PREVIOUS_IMAGE}"
                    try {
                        sh '''
                            IMAGE_REF=$IMAGE:$VERSION APP_ENV=prod APP_COLOR='#2da44e' PORT=8082 \
                                $COMPOSE -p tp3-8-prod up -d --wait
                            ./deploy/smoke-test.sh tp3-8-prod $VERSION
                        '''
                    } catch (err) {
                        if (env.PREVIOUS_IMAGE != 'none') {
                            echo "Smoke test KO : retour à ${env.PREVIOUS_IMAGE}"
                            sh '''
                                IMAGE_REF=$PREVIOUS_IMAGE APP_ENV=prod APP_COLOR='#2da44e' PORT=8082 \
                                    $COMPOSE -p tp3-8-prod up -d --wait
                            '''
                        }
                        error "Déploiement production échoué, rollback effectué"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} : version ${VERSION} déployée"
        }
        failure {
            echo "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} en échec"
            // Email Extension configuré sur le serveur SMTP local tp3.8-mailpit (boîte visible sur http://localhost:8025)
            emailext to: 'baptiste@legeni.local', subject: "Échec ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: "Le pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} (version ${VERSION}) a échoué.\nConsole : ${env.BUILD_URL}console"
        }
    }
}
