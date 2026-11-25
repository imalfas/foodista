
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'imalfas/foodista'
        SONAR_SERVER_NAME     = 'sonarqube-env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/imalfas/foodista.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                echo "Source code checked out successfully"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=foodista-key \
                            -Dsonar.projectName=foodista-project \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login \
                                -u "$DOCKER_USER" --password-stdin

                            docker build \
                                -t imalfas/foodista:${BUILD_NUMBER} \
                                -t imalfas/foodista:latest \
                                -f Dockerfile .

                            docker push imalfas/foodista:${BUILD_NUMBER}
                            docker push imalfas/foodista:latest

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${MANIFEST_BRANCH}"]],
                        userRemoteConfigs: [[
                            credentialsId: "${GIT_CREDENTIALS_ID}",
                            url: "${MANIFEST_REPO}"
                        ]]
                    ])

                    sh '''
                        sed -i "s|image:.*|image: imalfas/foodista:${BUILD_NUMBER}|" manifests/deployment.yaml

                        git config user.name "imalfas"
                        git config user.email "muhammadalfas9946@gmail.com"

                        git add manifests/deployment.yaml

                        if ! git diff --cached --quiet; then
                            git commit -m "Update foodista image to ${BUILD_NUMBER}"
                        else
                            echo "No manifest changes to commit"
                            exit 0
                        fi
                    '''

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${GIT_CREDENTIALS_ID}",
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {
                        sh '''
                            git -c credential.helper='!f() { echo "username=$GIT_USERNAME"; echo "password=$GIT_TOKEN"; }; f' push origin HEAD:main
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully! Docker image pushed and manifest updated.'
        }

        failure {
            echo 'Pipeline failed. Please check the console output.'
        }
    }
}

