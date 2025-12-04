pipeline {
  agent any
  environment {
    REPO_NAME = 'Abhiram303/Javawebapp-Docker_image-Pipeline'
    REMOTE_SERVER = 'your_server_IP'
    REMOTE_USER = 'ec2-user'
    APP_CONTAINER = 'javaApp'
    APP_PORT = '8081'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: '*/main']],
          userRemoteConfigs: [[url: 'https://github.com/Abhiram303/Javawebapp-Docker_image-Pipeline.git']]
        ])
      }
    }

    stage('Set metadata') {
      steps {
        script {
          GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          IMAGE_TAG = "${GIT_COMMIT_SHORT}-${env.BUILD_NUMBER}"
          env.IMAGE = "${REPO_NAME}:${IMAGE_TAG}"
        }
      }
    }

    stage('Build (Maven)') {
      steps {
        sh 'mvn -B -DskipTests=false clean package'
      }
      post {
        success {
          archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
      }
    }

    stage('Test') {
      steps {
        sh 'mvn test -B'
      }
    }

    stage('Build Docker image') {
      steps {
        sh "docker build -t ${IMAGE} ."
      }
    }

    stage('Authenticate & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE}
            docker logout
          """
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        // Use SSH key credential stored in Jenkins as 'aws-ssh-key' (private key)
        sshagent(credentials: ['aws-ssh-key']) {
          sh """
            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_SERVER} '
              set -e
              docker pull ${IMAGE}
              docker stop ${APP_CONTAINER} || true
              docker rm ${APP_CONTAINER} || true
              docker run --name ${APP_CONTAINER} -d -p ${APP_PORT}:${APP_PORT} --restart unless-stopped ${IMAGE}
            '
          """
        }
      }
    }

    stage('Post-deploy validation') {
      steps {
        // Basic HTTP probe — replace with proper healthcheck endpoint
        sh """
          sleep 6
          ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_SERVER} 'curl -f http://localhost:${APP_PORT}/ || exit 1'
        """
      }
      post {
        failure {
          // implement rollback, notifications or mark deployment failed
          echo "Post-deploy check failed. Manual rollback required (or implement automated rollback)."
        }
      }
    }
  } // stages

  post {
    always {
      cleanWs()
    }
    success {
      echo "Pipeline completed: pushed ${IMAGE}"
    }
    failure {
      echo "Pipeline failed. Check logs."
    }
  }
}
