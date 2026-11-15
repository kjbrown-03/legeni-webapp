// Projet final - chaîne CI/CD « du commit à la production » - Baptiste Kaldjob
// CI sur toutes les branches et PR ; push, déploiements et tag Git sur main uniquement.
// Credentials Jenkins utilisés (aucun secret dans ce fichier) :
//   registry-credentials (Docker Hub), kubeconfig (fichier), ansible-vault-pass (fichier), github (tag Git)
pipeline {
    agent any

    environment {
        REGISTRY_IMAGE = 'kaldjobbaptiste/legeni-webapp'
        APP_VERSION    = readFile('VERSION').trim()
        IMAGE_TAG      = "${APP_VERSION}-${env.BUILD_NUMBER}"      // ex. 1.1.0-37, jamais latest
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()                 // jamais deux déploiements en même temps
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    stages {
        stage('Contrôles qualité') {
            parallel {
                stage('Tests + couverture') {
                    agent { docker { image 'python:3.12-slim'; args '-e HOME=/tmp'; reuseNode true } }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q --retries 10 --timeout 120 -r requirements.txt -r requirements-dev.txt
                            mkdir -p reports
                            /tmp/venv/bin/pytest --junitxml=reports/junit.xml \
                                --cov=app --cov-report=xml:reports/coverage.xml --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit 'reports/junit.xml'
                            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                        }
                    }
                }
                stage('Lint Python') {
                    agent { docker { image 'python:3.12-slim'; args '-e HOME=/tmp'; reuseNode true } }
                    steps {
                        sh '''
                            python -m venv /tmp/venv-lint
                            /tmp/venv-lint/bin/pip install -q --retries 10 --timeout 120 flake8==7.1.1
                            mkdir -p reports
                            /tmp/venv-lint/bin/flake8 --output-file=reports/flake8.txt --tee .
                        '''
                    }
                    post { always { recordIssues(tools: [flake8(pattern: 'reports/flake8.txt')]) } }
                }
                stage('Lint Dockerfile') {
                    steps { sh 'docker run --rm -i hadolint/hadolint:v2.12.0 < Dockerfile' }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        dir('ansible') {
                            sh 'ansible-playbook --syntax-check provision.yml deploy.yml'
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$IMAGE_TAG -t $REGISTRY_IMAGE:$IMAGE_TAG .'
            }
        }

        stage('Scan Trivy') {
            steps {
                // quality gate : le pipeline s'arrête si une vulnérabilité CRITICAL corrigeable est trouvée
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
                        aquasec/trivy:0.57.1 image --timeout 15m --severity CRITICAL --ignore-unfixed --exit-code 1 \
                        $REGISTRY_IMAGE:$IMAGE_TAG
                '''
            }
        }

        stage('Push registre') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REG_USER', passwordVariable: 'REG_TOKEN')]) {
                    sh '''
                        echo "$REG_TOKEN" | docker login -u "$REG_USER" --password-stdin
                        docker push $REGISTRY_IMAGE:$IMAGE_TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when { branch 'main' }
            steps { deployer('staging') }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Déployer ${IMAGE_TAG} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            steps { deployer('prod') }
        }

        stage('Tag Git') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github',
                                                  usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        URL="https://$GH_USER:$GH_TOKEN@github.com/kjbrown-03/legeni-webapp.git"
                        if git ls-remote --tags "$URL" | grep -q "refs/tags/v$APP_VERSION$"; then
                            echo "Tag v$APP_VERSION déjà présent (même version redéployée)"
                        else
                            git -c user.name="Jenkins (Baptiste Kaldjob)" -c user.email="baptiste@legeni.local" \
                                tag -a "v$APP_VERSION" -m "Mise en production $IMAGE_TAG"
                            git push "$URL" "v$APP_VERSION"
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} : ${env.IMAGE_TAG ?: '?'}"
        }
        failure {
            echo "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} en échec"
            emailext to: 'baptiste@legeni.local',
                     subject: "Échec ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: "Le pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.IMAGE_TAG ?: 'image non construite'}) a échoué.\nConsole : ${env.BUILD_URL}console"
        }
    }
}

// Déploiement par Ansible (deploy.yml) : attend le rolling update, smoke test, rollback automatique en cas d'échec
def deployer(String cible) {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                     file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
        dir('ansible') {
            sh """
                export K8S_AUTH_KUBECONFIG="\$KUBECONFIG"
                ansible-playbook deploy.yml -e target_env=${cible} -e app_version=\$IMAGE_TAG \
                    --vault-password-file "\$VAULT_PASS"
            """
        }
    }
}
