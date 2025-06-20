pipeline {
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    agent any

    stages {
        stage('Set Build Name') {
            steps {
                script {
                    def branchNameParts = env.GIT_BRANCH.tokenize('/')

                    env.VARIABLE_GROUP = branchNameParts[1]
                    def version = branchNameParts[2]

                    def date = new Date().format("yyyyMMdd.HHmm")

                    def commitId = env.GIT_COMMIT ?: "unknown"

                    def buildName = "${version}.${date}.${commitId}"

                    env.REYNA_PETSHOP_BUILD_NUMBER = buildName

                    echo "REYNA_PETSHOP_BUILD_NUMBER: ${env.REYNA_PETSHOP_BUILD_NUMBER}"

                    currentBuild.displayName = "(${env.VARIABLE_GROUP}) ${buildName}"

                    echo "Build Name: ${currentBuild.displayName}"
                }
            }
        }

        stage('Load Environment Variables') {
            steps {
                script {
                    def credentialsId = "${env.VARIABLE_GROUP}.env"

                    withCredentials([file(credentialsId: credentialsId, variable: 'SECRET_ENV_FILE')]) {
                        def envFileContent = sh(script: "cat ${SECRET_ENV_FILE}", returnStdout: true).trim()

                        envFileContent.split('\n').each { line ->
                            def parts = line.split('=')

                            if (parts.length == 2) {
                                def key = parts[0].trim()
                                def value = parts[1].trim()

                                env[key] = value
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    env.DOCKER_IMAGE = "reyna-petshop"
                    env.DOCKER_TAG = "${env.REYNA_PETSHOP_BUILD_NUMBER}"

                    sh "docker build -t ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} -f ./Dockerfile ."
                }
            }
        }

        stage('Run Container') {
            steps {
                sh """
                    docker stop ${env.DOCKER_IMAGE} || true
                    docker rm ${env.DOCKER_IMAGE} || true

                    docker run -d \
                        --name ${env.DOCKER_IMAGE} \
                        -p 8000:8000 \
                        ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                """
            }
        }

        stage('Clean Old Images') {
            steps {
                sh """
                    docker images --format "{{.Repository}}:{{.Tag}}" | grep "^${env.DOCKER_IMAGE}:" | grep -v ":${env.DOCKER_TAG}" | xargs -r docker rmi
                """
            }
        }

        stage('Archive Artifacts') {
            steps {
                script {
                    archiveArtifacts allowEmptyArchive: true, artifacts: '*', onlyIfSuccessful: true
                }
            }
        }
    }

    post {
        success {
            echo "Build completed successfully"
        }
        failure {
            echo "Build failed"
        }
        always {
            cleanWs()
        }
    }
}
