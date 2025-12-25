
pipeline {
  agent {
  environment {
    NVD_API_KEY = credentials('nvd-api-key') // if stored in Jenkins credentials
  }
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }

  stages {

    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
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
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts(
                allowEmptyArchive: true,
                artifacts: 'target/dependency-check-report.html',
                fingerprint: true,
                onlyIfSuccessful: false
              )
              // dependencyCheckPublisher pattern: 'report.xml'
            }
          }
        }

      }
    }

    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
      }
    }

    stage('Build Docker Image') {
      agent { label 'kaniko' } // must match pod template label
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              -f `pwd`/Dockerfile \
              -c `pwd` \
              --destination=docker.io/ernest633/dso-demo:v1 \
              --cache=true \
              --skip-tls-verify
          '''
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        sh "echo done"
      }
    }

  }
}
