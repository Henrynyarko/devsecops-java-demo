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
            parallel(
                "Unit Tests": {
                    container('maven') {
                        sh 'mvn test'
                    }
                },
                "SCA": {
                    container('maven') {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            withCredentials([
                                string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')
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
                        // Archive report even if scan fails
                        archiveArtifacts(
                            allowEmptyArchive: true,
                            artifacts: 'target/dependency-check-report.html',
                            fingerprint: true
                        )
                    }
                },
                "OSS License Checker": {
                    container('licensefinder') {
                        sh 'ls -al'
                        sh '''#!/bin/bash --login
                            /bin/bash --login
                            rvm use default || true
                            gem install license_finder || true
                            license_finder || true
                        '''
                    }
                }
                // Add SASTs as another parallel branch if needed
            )
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
            steps {
                echo "Deployment to Dev completed"
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"
        }
    }
}
