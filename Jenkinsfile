pipeline {
  agent any
  options { timestamps(); skipDefaultCheckout(true) }

  environment {
    AWS_REGION = 'us-east-2'
    ACCOUNT_ID = '850924742604'
    ECR_REPO   = 'mydockerrepo'
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"
    REPO_URL   = 'https://github.com/Opey01/springboot-app.git'
    BRANCH     = 'main'
    DOCKERFILE = 'Dockerfilejenkins'
  }

  stages {
    stage('Prep (clean workspace)') { steps { deleteDir() } }

    stage('Checkout') {
      steps {
        git url: "${REPO_URL}", branch: "${BRANCH}", changelog: false, poll: false
      }
    }

    stage('Build (Maven)') {
      steps {
        // Build using root pom.xml. This creates target/springbootApp.jar
        sh 'mvn -B -DskipTests clean package'
      }
    }

    stage('Archive Jar') {
      steps { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true }
    }

    stage('Docker Build & Tag') {
      steps {
        script { env.ECR_URL = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}" }
        sh 'docker build -f ${DOCKERFILE} -t ${ECR_URL}:${IMAGE_TAG} .'
      }
    }

    // Uses Trivy via Docker (no host install required). Remove "|| true" to fail the build on findings.
    stage('Trivy Scan') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy:latest image --exit-code 1 --severity CRITICAL,HIGH \
          ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG} || true
        '''
      }
    }

    stage('ECR Login & Push') {
      steps {
        sh '''
          aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1 || \
            aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}

          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

          docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
        '''
      }
    }
  }

  post {
    always {
      sh 'docker logout ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com || true'
      cleanWs()
    }
  }
}
