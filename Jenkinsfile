pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'zuru07/bluegreen-sample'
        VERSION = '${BUILD_NUMBER}'
        BLUE_PORT = '3001'
        GREEN_PORT = '3002'
        ACTIVE_PORT_FILE = 'active-port.txt'
    }

    stages {
        stage('Build and Push Docker Image') {
            steps {
                script {
                    def customImage = docker.build("${DOCKER_IMAGE}:v${VERSION}")
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-cred') {
                        customImage.push()
                    }
                }
            }
        }

        stage('Determine Inactive Environment') {
            steps {
                script {
                    def activePort = "${BLUE_PORT}"  // Default to blue on first run
                    if (fileExists(env.ACTIVE_PORT_FILE)) {
                        activePort = readFile(env.ACTIVE_PORT_FILE).trim()
                    }
                    if (activePort == BLUE_PORT) {
                        env.INACTIVE_PORT = GREEN_PORT
                        env.ACTIVE_PORT = BLUE_PORT
                    } else {
                        env.INACTIVE_PORT = BLUE_PORT
                        env.ACTIVE_PORT = GREEN_PORT
                    }
                    writeFile file: env.ACTIVE_PORT_FILE, text: env.ACTIVE_PORT
                }
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                sh """
                    docker stop ${env.INACTIVE_PORT == BLUE_PORT ? 'blue-container' : 'green-container'} || true
                    docker rm ${env.INACTIVE_PORT == BLUE_PORT ? 'blue-container' : 'green-container'} || true
                    docker run -d --name ${env.INACTIVE_PORT == BLUE_PORT ? 'blue-container' : 'green-container'} -p ${env.INACTIVE_PORT}:3000 ${DOCKER_IMAGE}:${VERSION}
                """
            }
        }

        stage('Test Inactive Environment') {
            steps {
                sh "timeout /t 5"  // Wait 5 seconds for container to start (Windows)
                bat "curl -s http://localhost:${env.INACTIVE_PORT} >nul 2>&1 || exit /b 1"
                echo "Testing ${env.INACTIVE_PORT} environment"
            }
        }

        stage('Switch Traffic') {
            steps {
                bat "echo ${env.INACTIVE_PORT} > ${env.ACTIVE_PORT_FILE}"
                echo "Traffic switched to port ${env.INACTIVE_PORT}. Update your client to use http://localhost:${env.INACTIVE_PORT}."
            }
        }

        stage('Cleanup Old Environment') {
            steps {
                echo "Old env (port ${env.ACTIVE_PORT}) is now inactive."
            }
        }
    }

    post {
        failure {
            bat "echo 'Rollback not implemented; manually switch back to ${env.ACTIVE_PORT} if needed.'"
        }
    }
}