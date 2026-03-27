pipeline {
    agent any

    triggers {
        // Build on push and PRs
        pollSCM('H/5 * * * *')
    }

    environment {
        APP_NAME = 'my-web-app'
        REGISTRY = 'docker.io/myorg'
    }

    parameters {
        string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: 'Target deployment environment')
    }

    stages {
        stage('Print Build Info') {
            steps {
                // These reference PR/branch metadata that Jenkins exposes via env vars.
                // A naive migration might interpolate github.event context directly
                // into run: steps, which is a script injection vector.
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Change author: ${env.CHANGE_AUTHOR}"
                echo "PR Title: ${env.CHANGE_TITLE}"
                echo "Commit message: ${env.GIT_COMMIT_MESSAGE}"
                echo "Deploy env: ${params.DEPLOY_ENV}"
            }
        }

        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                }
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Tag image with branch name — another injection-prone pattern
                    // when branch name comes from untrusted input (e.g., fork PRs)
                    def tag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    sh "docker build -t ${REGISTRY}/${APP_NAME}:${tag} ."
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS'),
                    string(credentialsId: 'deploy-token', variable: 'DEPLOY_TOKEN')
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${REGISTRY}/${APP_NAME}:main-${BUILD_NUMBER}
                    '''
                    sh """
                        curl -X POST https://deploy.example.com/api/deploy \
                            -H "Authorization: Bearer ${DEPLOY_TOKEN}" \
                            -d '{"app": "${APP_NAME}", "env": "${params.DEPLOY_ENV}"}'
                    """
                }
            }
        }
    }

    post {
        failure {
            // Notification with PR title in the message — injection-prone when migrated
            echo "Build failed for PR: ${env.CHANGE_TITLE} by ${env.CHANGE_AUTHOR}"
        }
        success {
            echo "Build succeeded for ${env.BRANCH_NAME}"
        }
    }
}
