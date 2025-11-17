Deployment piepline code 
pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials("github-token")
    }

    parameters {
        string(name: "App_Name", description: "application name that needs to be deployed")
        string(name: "App_Version", description: "version of the application that needs to be deployed")
    }

    stages {

        stage("cloning repo") {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/mdshadab0/Kubernetes-ArgoCD.git']]
                )
            }
        }

        stage("updating image version in a file") {
            steps {
                sh """
                    echo "-------- Updating File Content --------"
                    cd "${params.App_Name}"
                    sed -i 's|image:.*|image: mdshadab0500/datastore:${params.App_Version}|' "${params.App_Name}.yaml"
                    echo "-------- File Content Updated Successfully --------"
                """
            }
        }

        stage("pushing code to github") {
            steps {
                sh """
                    echo "-------- Pushing Changes To GitHub --------"
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins"

                    git add .
                    git commit -am "docker image updated to ${params.App_Version}"
                    git push https://${GITHUB_TOKEN}@github.com/mdshadab0/Kubernetes-ArgoCD.git HEAD:main

                    echo "-------- Pushed Changes Successfully --------"
                """
            }
        }

    }
}
