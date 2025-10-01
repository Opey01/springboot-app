pipeline {
  agent any
  options { timestamps(); skipDefaultCheckout(true) }

  environment {
    AWS_REGION = 'us-east-2'
    ACCOUNT_ID = '850924742604'
    ECR_REPO   = 'mydockerrepo'
    REGISTRY   = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    IMAGE_TAG  = "build-${BUILD_NUMBER}"
    IMAGE_URI  = "${REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"

    POM_PATH        = 'MyAwesomeApp/pom.xml'
    DOCKERFILE_PATH = 'Dockerfilejenkins'
    BUILD_CONTEXT   = '.'

    AWS_CREDENTIALS_ID = ''   // set a Jenkins AWS creds ID here only if NOT using instance role
  }

  stages {
    stage('Prep (clean workspace)') { steps { deleteDir() } }

    stage('Checkout') {
      steps {
        // Explicitly use your GitHub repo + main branch
        git url: 'https://github.com/Opey01/springboot-app.git', branch: 'main'
      }
    }

    stage('Build (Maven)') {
      steps {
        sh 'mvn -f ${POM_PATH} -B -DskipTests clean package'
      }
    }

    stage('Archive Jar') {
      steps {
        archiveArtifacts artifacts: 'MyAwesomeApp/target/*.jar', fingerprint: true
      }
    }

    stage('Docker Build & Tag') {
      steps {
        sh '''
          docker build -f ${DOCKERFILE_PATH} -t ${ECR_REPO}:${IMAGE_TAG} ${BUILD_CONTEXT}
          docker tag ${ECR_REPO}:${IMAGE_TAG} ${IMAGE_URI}
          docker tag ${ECR_REPO}:${IMAGE_TAG} ${REGISTRY}/${ECR_REPO}:latest
        '''
      }
    }

    stage('Trivy Scan') {
      steps {
        sh '''
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v $WORKSPACE/.trivy-cache:/root/.cache/ \
            aquasec/trivy:latest image \
            --severity HIGH,CRITICAL \
            --no-progress \
            --exit-code 1 \
            ${IMAGE_URI}
        '''
      }
    }

    stage('ECR Login & Push') {
      steps {
        script {
          def pushToEcr = {
            sh '''
              aws ecr get-login-password --region "$AWS_REGION" \
                | docker login --username AWS --password-stdin "$REGISTRY"
              docker push "${IMAGE_URI}"
              docker push "${REGISTRY}/${ECR_REPO}:latest"
            '''
          }
          if (env.AWS_CREDENTIALS_ID?.trim()) {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDENTIALS_ID]]) {
              pushToEcr()
            }
          } else {
            pushToEcr() // instance role
          }
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout ${REGISTRY} || true'
      cleanWs()
    }
  }
}
