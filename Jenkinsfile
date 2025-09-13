pipeline {
    agent any
    
    environment {
        PROJECT_NAME = 'khoand-workshop2'
        REMOTE_HOST = '118.69.34.46'
        REMOTE_PORT = '3334'
        REMOTE_USER = 'newbie'
        REMOTE_PATH = '/usr/share/nginx/html/jenkins'
        LOCAL_CONTAINER = 'remote-host'
        LOCAL_PATH = '/usr/share/nginx/html/jenkins'
        WORKSPACE_NAME = 'khoand2'
        RELEASE_DATE = sh(script: 'date +%Y%m%d', returnStdout: true).trim()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "=== CHECKOUT STAGE ==="
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo "=== BUILD STAGE ==="
                sh 'npm install'
            }
        }
        
        stage('Lint/Test') {
            steps {
                echo "=== LINT/TEST STAGE ==="
                sh 'npm run test:ci'
            }
        }
        
        stage('Deploy') {
            parallel {
                stage('Deploy to Firebase') {
                    steps {
                        echo "=== DEPLOY TO FIREBASE ==="
                        withCredentials([string(credentialsId: 'firebase-token', variable: 'FIREBASE_TOKEN')]) {
                            sh '''
                                firebase deploy --token "$FIREBASE_TOKEN" --only hosting --project=${PROJECT_NAME}
                            '''
                        }
                    }
                }
                
                stage('Deploy to Local Container') {
                    steps {
                        echo "=== DEPLOY TO LOCAL CONTAINER ==="
                        script {
                            def releaseDir = "${LOCAL_PATH}/${WORKSPACE_NAME}/deploy/${RELEASE_DATE}"
                            
                            sh """
                                # Create directory structure in container
                                docker exec ${LOCAL_CONTAINER} mkdir -p ${releaseDir}
                                
                                # Copy essential files to container
                                docker cp index.html ${LOCAL_CONTAINER}:${releaseDir}/
                                docker cp 404.html ${LOCAL_CONTAINER}:${releaseDir}/
                                docker cp css ${LOCAL_CONTAINER}:${releaseDir}/
                                docker cp js ${LOCAL_CONTAINER}:${releaseDir}/
                                docker cp images ${LOCAL_CONTAINER}:${releaseDir}/
                                
                                # Update symlink and cleanup old releases
                                docker exec ${LOCAL_CONTAINER} bash -c "
                                    cd ${LOCAL_PATH}/${WORKSPACE_NAME}/deploy
                                    rm -f current
                                    ln -s ${RELEASE_DATE} current
                                    
                                    # Keep only 5 latest releases
                                    ls -1t | grep -E '^[0-9]{8}\$' | tail -n +6 | xargs -r rm -rf
                                "
                            """
                        }
                    }
                }
                
                stage('Deploy to Remote Server') {
                    steps {
                        echo "=== DEPLOY TO REMOTE SERVER ==="
                        script {
                            def releaseDir = "${REMOTE_PATH}/${WORKSPACE_NAME}/deploy/${RELEASE_DATE}"
                            
                            withCredentials([file(credentialsId: 'remote-ssh-key', variable: 'SSH_KEY')]) {
                                sh """
                                    # Create directory structure
                                    ssh -o StrictHostKeyChecking=no -i \$SSH_KEY -p ${REMOTE_PORT} ${REMOTE_USER}@${REMOTE_HOST} "
                                        mkdir -p ${releaseDir}
                                    "
                                    
                                    # Copy essential files only
                                    scp -o StrictHostKeyChecking=no -i \$SSH_KEY -P ${REMOTE_PORT} index.html ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                    scp -o StrictHostKeyChecking=no -i \$SSH_KEY -P ${REMOTE_PORT} 404.html ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                    scp -o StrictHostKeyChecking=no -i \$SSH_KEY -P ${REMOTE_PORT} -r css ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                    scp -o StrictHostKeyChecking=no -i \$SSH_KEY -P ${REMOTE_PORT} -r js ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                    scp -o StrictHostKeyChecking=no -i \$SSH_KEY -P ${REMOTE_PORT} -r images ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                    
                                    # Update symlink and cleanup old releases
                                    ssh -o StrictHostKeyChecking=no -i \$SSH_KEY -p ${REMOTE_PORT} ${REMOTE_USER}@${REMOTE_HOST} "
                                        cd ${REMOTE_PATH}/${WORKSPACE_NAME}/deploy
                                        rm -f current
                                        ln -s ${RELEASE_DATE} current
                                        
                                        # Keep only 5 latest releases
                                        ls -1t | grep -E '^[0-9]{8}\$' | tail -n +6 | xargs -r rm -rf
                                    "
                                """
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "=== DEPLOYMENT SUCCESS ==="
            slackSend(
                channel: '#lnd-2025-workshop',
                color: 'good',
                message: ":white_check_mark: *SUCCESS* - Workshop2 deployment completed!\n" +
                        "• *User:* ${env.BUILD_USER ?: 'System'}\n" +
                        "• *Job:* ${env.JOB_NAME}\n" +
                        "• *Build:* #${env.BUILD_NUMBER}\n" +
                        "• *Release:* ${RELEASE_DATE}\n" +
                        "• *Firebase:* https://khoand-workshop2.web.app\n" +
                        "• *Local Container:* Files deployed to ${LOCAL_CONTAINER}:${LOCAL_PATH}/${WORKSPACE_NAME}/deploy/current/\n" +
                        "• *Remote:* http://${REMOTE_HOST}/jenkins/${WORKSPACE_NAME}/deploy/current/"
            )
        }
        
        failure {
            echo "=== DEPLOYMENT FAILED ==="
            slackSend(
                channel: '#lnd-2025-workshop',
                color: 'danger',
                message: ":x: *FAILED* - Workshop2 deployment failed!\n" +
                        "• *User:* ${env.BUILD_USER ?: 'System'}\n" +
                        "• *Job:* ${env.JOB_NAME}\n" +
                        "• *Build:* #${env.BUILD_NUMBER}\n" +
                        "• *Error:* Check Jenkins console for details"
            )
        }
        
        always {
            echo "=== CLEANUP ==="
            cleanWs()
        }
    }
} 