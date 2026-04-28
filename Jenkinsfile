pipeline {
    agent any

    environment {
        APP_NAME     = "test-central-sonar-vault-01".toLowerCase()
        DOCKER_IMAGE = "sureshmavp1/${APP_NAME}"
        DOCKER_TAG   = "1.0.${BUILD_NUMBER}"
        IMAGE_TAG    = "${DOCKER_IMAGE}:${DOCKER_TAG}"
        GITOPS_REPO  = "https://github.com/Backstage-IDP-PoC/k8s-manifest.git"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Debug Variables') {
            steps {
                sh '''
                echo "APP_NAME=$APP_NAME"
                echo "DOCKER_IMAGE=$DOCKER_IMAGE"
                echo "IMAGE_TAG=$IMAGE_TAG"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Suresh SonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token1', variable: 'SONAR_AUTH_TOKEN')]) {
                        sh """
                        npx sonar-scanner \
                          -Dsonar.projectKey=${APP_NAME} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }


        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t ${IMAGE_TAG} .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                docker push ${IMAGE_TAG}
                '''
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([string(credentialsId: 'github-tokens', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                    rm -rf k8s-manifest
                    git clone https://${GITHUB_TOKEN}@github.com/Backstage-IDP-PoC/k8s-manifest.git
                    cd k8s-manifest

                    mkdir -p apps/${APP_NAME}
                    cp -r ../manifest/* apps/${APP_NAME}/

                    sed -i "s|\\${APP_NAME}|${APP_NAME}|g" apps/${APP_NAME}/*.yaml
                    sed -i "s|\\${DOCKER_IMAGE}|${IMAGE_TAG}|g" apps/${APP_NAME}/deployment.yaml

                    git config --global --add safe.directory '*'
                    git config user.email "jenkins@local"
                    git config user.name "jenkins"

                    git add .
                    git commit -m "Deploy ${APP_NAME} build ${BUILD_NUMBER}" || true
                    git push
                    '''
                }
            }
        }
    }

    post {
        always {
            sh "docker logout || true"
            sh "docker image prune -f || true"
        }
        success {
            echo "CI + CD pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
