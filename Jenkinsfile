pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: "1"))
    }

    environment {
        AWS_REGION = credentials('aws-region')
        ECR_REGISTRY = credentials('ecr-registry')
        ECR_REPO_NAME = credentials('ecr-repo-name')
        CI_SSH_KEY = credentials('github_auth')
        PROD_NAMESPACE = credentials('chioma-prod-namespace')
        STAGING_NAMESPACE = credentials('chioma-staging-namespace')
        PERSONAL_ACCESS_TOKEN = credentials('personal_access_token')
        GITHUB_USER = credentials('github_user')
        PROD_CLUSTER = credentials('prod-cluster')
        CLUSTER = credentials('cluster-name')
        DEVOPS_REPO_URL = credentials('devops-repo-url')
        REPO_NAME = sh(script: 'basename $(git remote get-url origin) .git', returnStdout: true).trim()
        SERVICE = "${REPO_NAME}"
        IMAGE_BUILD = "devops-products/build.sh"
        APP_DEPLOY = "devops-products/deploy.sh"
        DJANGO_SECRET = credentials('django-secret')
        BRANCH_NAME = sh(script: 'echo $GIT_BRANCH | sed "s|origin/||g"', returnStdout: true).trim()
        GITOPS_UPDATE = "devops-products/gitops-update.sh"
        CONFIG_REPO_URL = credentials('config-repo-url')
    }
        
    stages {
        stage('Grab DevOps Tools') {
            steps{
                script{
                    try {
                        def gitUser = env.CHANGE_AUTHOR ?: env.GIT_AUTHOR_NAME ?: sh(script: 'git log -1 --pretty=format:"%an"', returnStdout: true).trim()
                slackSend(color: '#FFFF00', 
                    message: """BUILD STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'
*Triggered By:* `${gitUser}`
- Be grateful for new beginnings!!!""".stripIndent()
                )
                        echo 'Checking and cleaning up existing resources'
                        sh '''
                            # Check and remove devops-products directory
                            if [ -d "devops-products" ]; then
                                echo "Found existing devops-products directory, removing it..."
                                rm -rf devops-products
                            fi
                            
                            # Check and remove tmp.txt file
                            if [ -f "tmp.txt" ]; then
                                echo "Found existing tmp.txt file, removing it..."
                                rm -f tmp.txt
                            fi
                            
                            # Check and remove password.txt file
                            if [ -f "password.txt" ]; then
                                echo "Found existing password.txt file, removing it..."
                                rm -f password.txt
                            fi
                            echo "LFG"
                            echo "Repository Name: ${REPO_NAME}"
                            echo "Service: ${SERVICE}"
                            chmod 600 "${CI_SSH_KEY}" || echo "Failed to set permissions on SSH key"
                            eval "$(ssh-agent -s)" > tmp.txt 2>&1 || echo "Failed to start SSH agent"
                            ssh-add "${CI_SSH_KEY}" > tmp.txt 2>&1 || echo "Failed to add SSH key"
                            git clone ${DEVOPS_REPO_URL}
                        '''
                    } 
                    catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        slackSend(color: '#FF0000', message: "Stage 'Grab DevOps Tools' FAILED: ${e.getMessage()} - Be grateful for the debugging opportunity!")
                        throw e
                    }
                }
            }
        }
        
        stage('Build Container Image') {
            steps {
                script {
                    try {
                        withCredentials([
                            aws(
                                credentialsId: 'aws-cred',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            )
                        ]) {
                            echo 'Building Container Image for ${SERVICE}...'
                            sh '''
                            chmod +x "${IMAGE_BUILD}"
                            "${IMAGE_BUILD}"
                            '''
                        }
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        slackSend(color: '#FF0000', message: "Stage 'Build Container Image' FAILED: ${e.getMessage()} - Be grateful for the debugging opportunity!")
                        throw e
                    }
                }
            }
        }

        stage('Deploy Application'){
            steps{
                script{
                    try {
                        def BRANCH_NAME = sh(script: 'echo $GIT_BRANCH | sed "s|origin/||g"', returnStdout: true).trim()
                        echo "Current branch is: ${BRANCH_NAME}"
                        
                        if (BRANCH_NAME.trim() == 'main') {
                            echo "Setting NAMESPACE to PROD_NAMESPACE"
                            NAMESPACE = "${PROD_NAMESPACE}"
                            // SECRET = "${PROD_SECRET}"
                        } else if (BRANCH_NAME.trim() == 'staging') {
                            echo "Setting NAMESPACE to STAGING_NAMESPACE"
                            NAMESPACE = "${STAGING_NAMESPACE}"
                            // NAMESPACE = "${TEST_NAMESPACE}"

                            // SECRET = "${STAGING_SECRET}"
                        } else {
                            error "Deployment is only allowed from main or staging branches"
                        }
                        
                        // Set cluster based on branch
                        if (BRANCH_NAME.trim() == 'main') {
                            CLUSTER_NAME = "${PROD_CLUSTER}"
                        } else if (BRANCH_NAME.trim() == 'staging') {
                            CLUSTER_NAME = "${PROD_CLUSTER}"
                        } else {
                            error "Deployment is only allowed from main or staging branches"
                        }
                        
                        echo "Deploying to cluster: ${CLUSTER_NAME}"
                        
                        withCredentials([
                            aws(
                                credentialsId: 'aws-cred',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            )
                        ]) {
                            echo "Deploying Application to ${NAMESPACE}..."
                            withEnv(["NAMESPACE=${NAMESPACE}", "CLUSTER_NAME=${CLUSTER_NAME}"]) {
                                sh """
                                chmod +x "${APP_DEPLOY}"
                                "${APP_DEPLOY}"
                                """
                            }
                        }
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        slackSend(color: '#FF0000', message: "Stage 'Deploy Application' FAILED: ${e.getMessage()} - Be grateful for the debugging opportunity!")
                        throw e
                    }
                }
            }
        }

         // stage('Update GitOps Config'){
        //     steps{
        //         script{
        //             try {
        //                 def BRANCH_NAME = sh(script: 'echo $GIT_BRANCH | sed "s|origin/||g"', returnStdout: true).trim()
        //                 echo "Current branch is: ${BRANCH_NAME}"
                        
        //                 echo "Updating GitOps config repository..."
                        
        //                 withEnv(["BRANCH_NAME=${BRANCH_NAME}"]) {
        //                     sh '''
        //                     chmod +x "${GITOPS_UPDATE}"
        //                     "${GITOPS_UPDATE}"
        //                     '''
        //                 }
                        
        //             } catch (Exception e) {
        //                 currentBuild.result = 'FAILURE'
        //                 slackSend(color: '#FF0000', message: "Stage 'Update GitOps Config' FAILED: ${e.getMessage()} - Be grateful for the debugging opportunity!")
        //                 throw e
        //             }
        //         }
        //     }
        // }

        stage('Clean Up'){
            steps{
                script{  
                    try {
                        echo 'Cleaning Up Unused Resources'
                        sh '''
                        pwd
                        rm -rf devops-products tmp.txt
                        ls 
                        '''
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        slackSend(color: '#FF0000', message: "Stage 'Clean Up' FAILED: ${e.getMessage()} - Be grateful for the debugging opportunity!")
                        throw e
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                slackSend(
                    color: 'good',
                    message: """
                        :white_check_mark: *Build Successful! Be grateful!*
                        *Job:* `${env.JOB_NAME}`
                        *Build Number:* `${env.BUILD_NUMBER}`
                        *Branch:* `${env.GIT_BRANCH}`
                        *Commit Hash:* `${commitHash}`
                        *Commit Message:* `${commitMessage}`
                        *Duration:* ${currentBuild.durationString}
                    """.stripIndent()
                )
            }
        }
        failure {
            script {
                def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                slackSend(
                    color: 'danger',
                    message: """
                        :x: *Build Failed! Be grateful for the learning opportunity!*
                        *Job:* `${env.JOB_NAME}`
                        *Build Number:* `${env.BUILD_NUMBER}`
                        *Branch:* `${env.GIT_BRANCH}`
                        *Commit Hash:* `${commitHash}`
                        *Commit Message:* `${commitMessage}`
                        *Duration:* ${currentBuild.durationString}
                        *Console Output:* ${env.BUILD_URL}console
                    """.stripIndent()
                )
            }
        }
        unstable {
            script {
                def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                slackSend(
                    color: 'warning',
                    message: """
                        :warning: *Build Unstable! Be grateful for the challenge!*
                        *Job:* `${env.JOB_NAME}`
                        *Build Number:* `${env.BUILD_NUMBER}`
                        *Branch:* `${env.GIT_BRANCH}`
                        *Commit Hash:* `${commitHash}`
                        *Commit Message:* `${commitMessage}`
                        *Duration:* ${currentBuild.durationString}
                    """.stripIndent()
                )
            }
        }
        always {
            script {
                def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                slackSend(
                    color: '#439FE0',
                    message: """
                        :bell: *Build Finished! Be grateful for another day of coding!*
                        *Job:* `${env.JOB_NAME}`
                        *Build Number:* `${env.BUILD_NUMBER}`
                        *Status:* ${currentBuild.result}
                        *Commit Hash:* `${commitHash}`
                        *Commit Message:* `${commitMessage}`
                    """.stripIndent()
                )
            }
        }
    }
}