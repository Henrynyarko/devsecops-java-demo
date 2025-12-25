pipeline {
    agent {
        kubernetes {
            yamlFile 'build-agent.yaml'
            defaultContainer 'maven'
            idleMinutes 1
        }
    }

    stages {

        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn compile'
                }
            }
        }

        stage('Static Analysis') {
            parallel {

                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                }

                stage('SCA') {
                    steps {
                        container('maven') {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                // Use withCredentials to properly inject NVD_API_KEY inside container
                                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                                    sh '''
                                        mvn org.owasp:dependency-check-maven:check \
                                        -Dnvd.api.key=$NVD_API_KEY
                                    '''
                                }
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts(
                                allowEmptyArchive: true,
                                artifacts: 'target/dependency-check-report.html',
                                fingerprint: true
                            )
                        }
                    }
                }
            }
        }

        stage('Package') {
            steps {
                container('maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }

        stage('Build Docker Image') {
            agent {
                kubernetes {
                    yamlFile 'build-agent.yaml'
                    defaultContainer 'kaniko'
                }
            }
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor \
                          -f Dockerfile \
                          -c . \
                          --destination=docker.io/ernest633/dso-demo:v1 \
                          --cache=true \
                          --skip-tls-verify
                    '''
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                echo "Deployment to Dev completed"
            }
        }
    }
}
