pipeline {
    agent any 

    environment {
        GCP_PROJECT_ID         = "neon-airway-450022-i1"
        GCP_REGION             = "us-central1"
        ARTIFACT_REGISTRY_REPO = "container-repository-gemini-jvp" 
        CLOUD_RUN_SERVICE_NAME = "gemini-angular-app"
        ENVIRONMENT_NAME       = "dev"
        
        GIT_COMMIT_SHORT       = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        IMAGE_TAG              = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REGISTRY_REPO}/${CLOUD_RUN_SERVICE_NAME}:${GIT_COMMIT_SHORT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Push Image') {
            agent {
                docker {
                    image 'google/cloud-sdk:stable'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --user root'
                }
            }

            steps {
                sh '''
                    apt-get update
                    apt-get install -y docker.io

                    gcloud config set project ${GCP_PROJECT_ID}

                    gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet

                    docker build -t ${IMAGE_TAG} .

                    docker push ${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to Cloud Run') {
            agent { 
                docker { 
                    image 'google/cloud-sdk:stable'
                    args '--user root'
                } 
            }
            steps {
                sh """
                gcloud run deploy ${env.CLOUD_RUN_SERVICE_NAME}-${env.ENVIRONMENT_NAME} \
                    --image ${env.IMAGE_TAG} \
                    --region ${GCP_REGION} \
                    --platform managed \
                    --allow-unauthenticated \
                    --set-env-vars="ENVIRONMENT_NAME=${env.ENVIRONMENT_NAME}"
                """
            }
        }
    }

    post {
        always {
            script {
                if (env.IMAGE_TAG) {
                    sh "docker rmi ${env.IMAGE_TAG} || true"
                }
            }
        }
    }
}