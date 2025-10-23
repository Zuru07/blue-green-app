pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'zuru07/bluegreen-sample'
        VERSION = 'v${BUILD_NUMBER}' 
        KUBECTL = 'kubectl'
        ACTIVE_ENV = 'blue'
    }

    stages {
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-cred') {
                        docker.image("${DOCKER_IMAGE}:${VERSION}").push()
                    }
                }
            }
        }

        stage('Determine Inactive Environment') {
            steps {
                script {
                    def currentSelector = sh(script: "${KUBECTL} get svc nodejs-service -o jsonpath='{.spec.selector.env}'", returnStdout: true).trim()
                    if (currentSelector == 'blue') {
                        env.INACTIVE_ENV = 'green'
                    } else {
                        env.INACTIVE_ENV = 'blue'
                    }
                    env.ACTIVE_ENV = currentSelector
                }
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh """
                        ${KUBECTL} apply -f k8s/deployment-${env.INACTIVE_ENV}.yaml --record
                        ${KUBECTL} set image deployment/nodejs-${env.INACTIVE_ENV} nodejs=${DOCKER_IMAGE}:${VERSION}
                        ${KUBECTL} rollout status deployment/nodejs-${env.INACTIVE_ENV}
                    """
                }
            }
        }

        stage('Test Inactive Environment') {
            steps {
                sh "echo 'Testing ${env.INACTIVE_ENV} environment'"
                // Add tests, e.g., curl http://nodejs-${env.INACTIVE_ENV}:3000 (adjust for your setup)
            }
        }

        stage('Switch Traffic') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "${KUBECTL} patch svc nodejs-service -p '{\"spec\":{\"selector\":{\"env\":\"${env.INACTIVE_ENV}\"}}}'"
                }
            }
        }

        stage('Cleanup Old Environment') {
            steps {
                echo "Old env (${env.ACTIVE_ENV}) is now inactive and ready for next deployment."
            }
        }
    }

    post {
        failure {
            withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "${KUBECTL} patch svc nodejs-service -p '{\"spec\":{\"selector\":{\"env\":\"${env.ACTIVE_ENV}\"}}}'"
            }
        }
    }
}