pipeline {

    environment {
        ARGO_SERVER = '34.122.47.13:32100'
    }

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
                                withCredentials([
                                    string(
                                        credentialsId: 'nvd-api-key',
                                        variable: 'NVD_API_KEY'
                                    )
                                ]) {
                                    sh '''
                                        echo "Using NVD API Key"
                                        mvn org.owasp:dependency-check-maven:check \
                                          -Dnvd.api.key=$NVD_API_KEY \
                                          -Dformat=HTML \
                                          -DoutputDirectory=target \
                                          -DfailOnError=false
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

        /*
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
        */

        stage('Deploy to Dev Environment') {
            environment {
                AUTH_TOKEN = credentials('argocd-jenkins-deployer-token')
            }
            steps {
                container('docker-tools') {

                    sh '''
                        docker run -t schoolofdevops/argocd-cli \
                          argocd app sync dso-demo \
                          --insecure \
                          --server $ARGO_SERVER \
                          --auth-token $AUTH_TOKEN
                    '''

                    sh '''
                        docker run -t schoolofdevops/argocd-cli \
                          argocd app wait dso-demo \
                          --health \
                          --timeout 300 \
                          --insecure \
                          --server $ARGO_SERVER \
                          --auth-token $AUTH_TOKEN
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"
        }
    }
}
