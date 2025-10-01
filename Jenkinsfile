pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "laly9999/secure-app"
        IMAGE_TAG    = "${BUILD_NUMBER}"   // 👈 this must be declared here
        REGISTRY_CREDENTIALS = credentials('dockerhub-credentials')
        SLACK_WEBHOOK        = credentials('slack-webhook')

        GCP_PROJECT  = "x-object-472022-q2"
        GCP_REGION     = "us-east4"
        GKE_CLUSTER  = "demo-autopilot"
    }

    stages {
        stage('Build') {
            steps {
                dir('app') {
                    sh 'npm install'
                }
            }
        }

       stage('Run SonarQube') {
            environment {
                scannerHome = tool 'sonar-scan'
            }
            steps {
                dir('app') {   // 👈 run commands inside app/
                    withSonarQubeEnv('MySonar') {
                        sh '''
                            # Run SonarQube analysis
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=secure-app \
                              -Dsonar.sources=. \
                        '''
                    }
                }
            }
        }

        stage('Secrets Detection - GitLeaks') {
            steps {
                sh "gitleaks detect --source . --no-banner --redact"
            }
        }


        stage('Docker Build') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$IMAGE_TAG ./app'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --exit-code 1 --severity CRITICAL $DOCKER_IMAGE:$IMAGE_TAG"
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    echo $REGISTRY_CREDENTIALS_PSW | docker login -u $REGISTRY_CREDENTIALS_USR --password-stdin
        
                    # Push with the correct tag (BUILD_NUMBER)
                    docker push $DOCKER_IMAGE:$IMAGE_TAG
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('helm-chart') {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh """
                            export USE_GKE_GCLOUD_AUTH_PLUGIN=True
                            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                            gcloud config set project $GCP_PROJECT
                        
                            kubectl apply -f ../k8s/deployment.yaml
                            kubectl apply -f ../k8s/service.yaml
                            kubectl set image deployment/secure-app secure-app=laly9999/secure-app:${BUILD_NUMBER}
                            kubectl rollout status deployment/secure-app

                        """
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                slackSend(
                    channel: '#devops-project',
                    message: "✅ Jenkins: Deployment successful for $DOCKER_IMAGE:$IMAGE_TAG"
                )
            }
        }
        failure {
            echo "Deployment failed ❌ → Rolling back..."
            withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                sh """
                  export USE_GKE_GCLOUD_AUTH_PLUGIN=True
                  gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                  gcloud config set project $GCP_PROJECT
                  # Undo the last deployment rollout
                  kubectl rollout undo deployment/secure-app
                  
                """
            }
            script {
                slackSend(
                    channel: '#devops-project',
                    message: "❌ Jenkins: Deployment FAILED for $DOCKER_IMAGE:$IMAGE_TAG. Rolled back to previous version."
                )
            }
        }
    }
}
