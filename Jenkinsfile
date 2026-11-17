pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'muhammedsufiyan/topology'

        SONAR_SERVER_NAME     = 'sonarqube-env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'

        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/suficapz/clone-test.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                echo "Source code checked out by Jenkins SCM"
                sh 'pwd && ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=topology-key \
                            -Dsonar.projectName=topology-project \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Build & Post Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                            docker build \
                                -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                                -t ${DOCKER_IMAGE}:latest \
                                -f Dockerfile .

                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {
                    git branch: "${MANIFEST_BRANCH}",
                        credentialsId: "${GIT_CREDENTIALS_ID}",
                        url: "${MANIFEST_REPO}"

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${GIT_CREDENTIALS_ID}",
                            usernameVariable: 'GH_USER',
                            passwordVariable: 'GH_TOKEN'
                        )
                    ]) {
                        sh """
                            sed -i '/name: topology/{n;s|image:.*|image: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}|}' manifests/deployment.yaml

                            git config user.name "suficapz"
                            git config user.email "sufiyanathikal@gmail.com"

                            git add manifests/deployment.yaml

                            if ! git diff --cached --quiet; then
                                git commit -m "chore(manifest): update topology image tag to ${env.BUILD_NUMBER} [skip ci]"

                                git remote set-url origin https://\${GH_USER}:\${GH_TOKEN}@github.com/suficapz/clone-test.git

                                git push origin ${MANIFEST_BRANCH}
                            else
                                echo "Manifest image is already up to date."
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully! Topology image pushed and manifest updated.'
        }

        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }
    }
}
