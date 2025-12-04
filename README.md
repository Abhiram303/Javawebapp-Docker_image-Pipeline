# Jenkinsfile (declarative, safer, production-minded) for docker image build, test, archieve, deploy to destination
Notes:
- Use `withCredentials` to access DockerHub and SSH keys so they aren’t printed into logs.
- Tag images with Git commit SHA and `BUILD_NUMBER` for traceability
- Archive artifacts and fail fast on tests.
- Use `sshagent` with an SSH credential ID for EC2 deploys.
- Add a simple health check/rollback placeholder.

```bash
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
```
> Must Noted points on the Jenkinsfile: the code block above is neutral and ready to paste. Replace credential IDs with ones you create in Jenkins

# Dockerfile (example for a Spring Boot or fat-jar app)

```dockerfile
# Build image not needed if Jenkins builds the jar. This Dockerfile assumes jar is present.
FROM eclipse-temurin:17-jre-alpine
ARG JAR_FILE=target/*.jar
WORKDIR /app
COPY ${JAR_FILE} app.jar
ENV JAVA_OPTS="-Xms256m -Xmx512m"
EXPOSE 8081
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8081/actuator/health || exit 1
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
```
> If you have a build stage (multi-stage docker build) you can compile inside the image. But Jenkins already runs Maven — keep Dockerfile simple and fast.


