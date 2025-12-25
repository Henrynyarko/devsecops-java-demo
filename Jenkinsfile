pipeline {
  // Top-level agent for the default container (Maven)
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }

  // Top-level environment variables
  environment {
    NVD_API_KEY = credentials('nvd-api-key')   // For OWASP Dependency Check
    GITHUB_TOKEN = credentials('github-pat')  // Your GitHub PAT
  }

  stages {

    stage('Checkout') {
      steps {
        // Checkout with GitHub PAT
        git branch: 'main', url: 'https://github.com/Henrynyarko/devsecops-java-demo', credentialsId: 'github-pat'
      }
    }

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
                sh '''
                  mvn org.owasp:dependency-check-maven:check \
                  -Dnvd.api.key=$NVD_API_KEY
                '''
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
      // Custom agent for Kaniko
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
        echo "Deployment to dev completed"
      }
    }

  }

  post {
    success {
      echo "Pipeline completed successfully"
    }
    failure {
      echo "Pipeline failed, check logs!"
    }
  }
}
