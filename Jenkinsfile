pipeline {
    agent any

    parameters {
        string(name: "App_Version", defaultValue: "v1.0", description: "Provide Application Version")
    }

    environment {
        AWS_REGION = "ap-south-1"
    }

    stages {

        stage("Repo Clone") {
            steps {
                checkout scmGit(
                    branches: [[name: '*/master']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/mdshadab0/DataStore.git']]
                )
            }
        }

        stage("Maven Build") {
            steps {
                sh '''
                    echo "-------- Building Application --------"
                    mvn clean package
                    echo "------- Application Built Successfully --------"
                '''
            }
        }

        stage("Maven Test") {
            steps {
                sh '''
                    echo "-------- Executing Testcases --------"
                    mvn test
                    echo "-------- Testcases Execution Complete --------"
                '''
            }
        }

        stage("Artifact Store") {
            steps {
                sh '''
                    echo "-------- Pushing Artifacts To S3 --------"
                    aws s3 cp ./target/*.jar s3://datastore-artefact-store/ --region ${AWS_REGION}
                    echo "-------- Pushing Artifacts To S3 Completed --------"
                '''
            }
        }

        stage("Docker Image Build") {
            steps {
                sh '''
                    echo "-------- Building Docker Image --------"
                    docker build -t datastore:${App_Version} .
                    echo "-------- Image Successfully Built --------"
                '''
            }
        }

        stage("Docker Image Scan") {
            steps {
                sh '''
                    echo "-------- Scanning Docker Image Using Trivy --------"
                    trivy image datastore:${App_Version}
                    echo "-------- Docker Image Scan Complete --------"
                '''
            }
        }

        stage("Docker Image Tag") {
            steps {
                sh '''
                    echo "-------- Tagging Docker Image --------"
                    docker tag datastore:${App_Version} mdshadab0500/datastore:${App_Version}
                    echo "-------- Tagging Docker Image Completed --------"
                '''
            }
        }

        stage("Login & Push Docker Image") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "-------- Logging To DockerHub --------"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        echo "-------- DockerHub Login Successful --------"

                        echo "-------- Pushing Docker Image To DockerHub --------"
                        docker push mdshadab0500/datastore:${App_Version}
                        echo "-------- Docker Image Pushed Successfully --------"
                    '''
                }
            }
        }

        stage("Cleanup") {
            steps {
                sh '''
                    echo "-------- Cleaning Up Jenkins Machine --------"
                    docker image prune -a -f
                    echo "-------- Clean Up Successful --------"
                '''
            }
        }

        stage("Deployment Acceptance") {
            steps {
                input 'Trigger Down Stream Job'
            }
        }

        stage("Triggering Deployment Job") {
            steps {
                build job: "deployment",
                parameters: [
                    string(name: "App_Name", value: "datastore-deploy"),
                    string(name: "App_Version", value: "${params.App_Version}")
                ]
            }
        }
    }

    post {
        success {
            slackSend channel: '#alerting',
                      color: 'good',
                      message: "✔️ SUCCESS: `${env.JOB_NAME} #${env.BUILD_NUMBER}` completed successfully."
        }

        failure {
            slackSend channel: '#alerting',
                      color: 'danger',
                      message: "❌ FAILURE: `${env.JOB_NAME} #${env.BUILD_NUMBER}` failed.\nCheck console output: ${env.BUILD_URL}"
        }

        unstable {
            slackSend channel: '#alerting',
                      color: 'warning',
                      message: "⚠️ UNSTABLE: `${env.JOB_NAME} #${env.BUILD_NUMBER}` returned unstable status."
        }

        aborted {
            slackSend channel: '#alerting',
                      color: '#808080',
                      message: "⛔ ABORTED: `${env.JOB_NAME} #${env.BUILD_NUMBER}` was aborted."
        }

        always {
            slackSend channel: '#alerting',
                      color: '#439FE0',
                      message: "ℹ️ Build finished: `${env.JOB_NAME} #${env.BUILD_NUMBER}`"
        }
    }
}
