pipeline {
  agent any
  tools {
    jdk 'java-21.0.11'
    maven 'Apache_Maven_3.9.9'
  }
  environment {
    SONAR_SCANNER_HOME = tool 'sonar7'
    IMAGE_NAME = "java-app"
    IMAGE_TAG = "${BUILD_NUMBER}"
    GCP_PROJECT_ID = "focal-dock-440200-u5"
    FULL_IMAGE_NAME = "us-docker.pkg.dev/${GCP_PROJECT_ID}/java-app-repo-02/${IMAGE_NAME}:${IMAGE_TAG}"
    SERVICE_NAME = "java-app-service"
    REGION = "us-central1"
  }
  stages {
    stage('Initialize Pipeline') {
      steps {
        echo 'Initializing Pipeline ...'
        sh 'java -version'
        sh 'mvn -version'
      }
    }
    stage('Checkout GitHub Codes') {
      steps {
        echo 'Checking out GitHub Codes ...'
        git branch: 'main', url: 'https://github.com/iQuantC/Jenkins_GCP_CloudRun.git', credentialsId: 'jenkins-gcp'
      }
    }
    stage('Maven Build') {
      steps {
        echo 'Building Java App with Maven'
        sh 'mvn clean package'
      }
    }
    stage('JUnit Test of Java App') {
      steps {
        echo 'JUnit Testing'
        sh 'mvn test'
      }
    }
    stage('SonarQube Analysis') {
      steps {
        echo 'Running Static Code Analysis with SonarQube'
        withCredentials([string(credentialsId: 'sonartoken', variable: 'sonarToken')]) {
          withSonarQubeEnv('sonar') {
            sh """
              ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                -Dsonar.projectKey=jenkinsgcp \
                -Dsonar.sources=. \
                -Dsonar.host.url=http://172.18.0.3:9000 \
                -Dsonar.java.binaries=target/classes \
                -Dsonar.login=$sonarToken
            """
          }
        }
      }
    }
    stage('Build & Tag Docker Image') {
      steps {
        echo 'Building the Java App Docker Image'
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
      }
    }
    stage('Authenticate with GCP, Tag & Push to Artifact Registry') {
      steps {
        withCredentials([file(credentialsId: 'gcpjmsa', variable: 'gcpCred')]) {
          withEnv(["GOOGLE_APPLICATION_CREDENTIALS=$gcpCred"]) {
            sh '''
              gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
              gcloud config set project $GCP_PROJECT_ID
              gcloud auth configure-docker us-docker.pkg.dev --quiet
            '''
            // assume repository already exists; do not create per-build
            sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}"
            sh "docker push ${FULL_IMAGE_NAME}"
          }
        }
      }
    }
    stage('Deploy to Cloud Run') {
      steps {
        withCredentials([file(credentialsId: 'gcpjmsa', variable: 'gcpCred')]) {
          withEnv(["GOOGLE_APPLICATION_CREDENTIALS=$gcpCred"]) {
            sh '''
              gcloud run deploy $SERVICE_NAME \
                --image=$FULL_IMAGE_NAME \
                --region=$REGION \
                --platform=managed \
                --allow-unauthenticated \
                --port=8090 \
                --memory=512Mi \
                --quiet
            '''
          }
        }
      }
    }
    stage('Get Cloud Run Service URL') {
      steps {
        withCredentials([file(credentialsId: 'gcpjmsa', variable: 'gcpCred')]) {
          withEnv(["GOOGLE_APPLICATION_CREDENTIALS=$gcpCred"]) {
            sh '''
              SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
                --platform managed \
                --region $REGION \
                --format="value(status.url)")
              echo "Service URL: $SERVICE_URL"
            '''
          }
        }
      }
    }
  }
}
