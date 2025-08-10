pipeline {
    agent any

    environment {
        IMAGE_NAME            = 'rani19/hello-web'
        TAG                   = "build-${env.BUILD_NUMBER}"
        DEV_REPO_URL          = 'https://github.com/RaniSaed/hello-web.git'
        CONFIG_REPO_URL       = 'https://github.com/RaniSaed/hello-web-config.git'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        CONFIG_FILE_PATH      = 'k8s/deployment.yaml'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 20, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Clone Dev Repo') {
            steps {
                dir('dev') {
                    git url: "${DEV_REPO_URL}", branch: 'main'
                }
            }
        }

        stage('Clone Config Repo') {
            steps {
                dir('config') {
                    git url: "${CONFIG_REPO_URL}", branch: 'main'
                }
            }
        }

        stage('Check for Changes') {
            steps {
                dir('dev') {
                    script {
                        def rc = sh(
                          script: '''
                            set -e
                            if [ "$(git rev-list --count HEAD)" -gt 1 ]; then
                              git diff --name-only HEAD~1 HEAD | grep -E '^(Dockerfile|index.html|Jenkinsfile)$' >/dev/null
                            else
                              exit 0
                            fi
                          ''',
                          returnStatus: true
                        )
                        if (rc != 0) {
                            echo "No relevant app changes detected. Aborting."
                            currentBuild.result = 'ABORTED'
                            error("No changes in app files")
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('dev') {
                    script {
                        echo "Building image ${IMAGE_NAME}:${TAG}"
                        docker.build("${IMAGE_NAME}:${TAG}", ".")
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                            docker.image("${IMAGE_NAME}:${TAG}").push()
                            docker.image("${IMAGE_NAME}:${TAG}").push('latest')
                        }
                    }
                }
            }
        }

        stage('Update Deployment YAML') {
            steps {
                dir('config') {
                    script {
                        def newImage = "${IMAGE_NAME}:${TAG}"
                        sh '''
                          sed -i.bak -E "s|^\\s*image:\\s*.*|image: ''' + "${IMAGE_NAME}:${TAG}" + '''|" ''' + "${CONFIG_FILE_PATH}" + '''
                          rm -f ''' + "${CONFIG_FILE_PATH}" + '''.bak
                          grep -n "image:" ''' + "${CONFIG_FILE_PATH}" + ''' || true
                        '''
                    }
                }
            }
        }

        stage('Commit & Push Changes') {
            steps {
                dir('config') {
                    withCredentials([string(credentialsId: 'github-creds', variable: 'GIT_TOKEN')]) {
                        sh '''
                          set -e
                          git config user.email "rani.saed19@gmail.com"
                          git config user.name  "Rani Saed (CI/CD)"
                          git add ''' + "${CONFIG_FILE_PATH}" + '''
                          if git diff --cached --quiet; then
                            echo "No changes to commit."
                          else
                            git commit -m "ci: update hello-web image to ''' + "${IMAGE_NAME}:${TAG}" + '''"
                            git push https://x-access-token:$GIT_TOKEN@github.com/RaniSaed/hello-web-config.git HEAD:main
                          fi
                        '''
                    }
                }
            }
        }
    }
}
