pipeline {
    agent any

    environment {
        ECR_REPO = "816039038499.dkr.ecr.ap-south-1.amazonaws.com/generic-app"
        AWS_DEFAULT_REGION = "ap-south-1"
    }

    triggers {
        pollSCM('H/15 * * * *') // Poll every 15 mins
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning repository..."
                checkout scm
                sh 'git fetch origin'
            }
        }

        stage('Git Diff Check') {
    steps {
        script {
            // Get latest commit hash from remote
            def latestCommit = sh(script: "git rev-parse origin/main", returnStdout: true).trim()

            // Read last successful commit hash (if exists)
            def lastCommitFile = "${env.WORKSPACE}/.last_successful_commit"
            def lastCommit = fileExists(lastCommitFile) ? readFile(lastCommitFile).trim() : ""

            if (lastCommit == latestCommit) {
                echo "No code changes detected — skipping build."
                currentBuild.result = 'ABORTED'
                error("Aborting build: No code changes")
            } else {
                echo "Code changes detected — proceeding with build."
                writeFile file: lastCommitFile, text: latestCommit
            }
        }
    }
}



        stage('Build Image') {
            when {
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                echo "Building Docker image..."
                sh "docker build -t generic-app:${BUILD_NUMBER} ."
            }
        }

        stage('Tag and Push') {
            when {
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                withCredentials([aws(credentialsId: 'ecr-creds', region: "${AWS_DEFAULT_REGION}")]) {
                    script {
                        try {
                            sh '''
                                aws ecr get-login-password --region ${AWS_DEFAULT_REGION} \
                                    | docker login --username AWS --password-stdin ${ECR_REPO}

                                docker tag generic-app:${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}
                                docker tag generic-app:${BUILD_NUMBER} ${ECR_REPO}:latest

                                docker push ${ECR_REPO}:${BUILD_NUMBER}
                                docker push ${ECR_REPO}:latest
                            '''
                        } catch (err) {
                            echo "Push Validation Error: ${err.getMessage()} - Tag conflict?"
                            currentBuild.result = 'ABORTED'
                            error("Aborting build due to ECR push validation failure")
                        }
                    }
                }
            }
        }
    }
}

