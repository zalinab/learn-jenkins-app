pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '1d9d533b-8b27-476c-bc64-edf9935bb765'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }
    stages {
        stage('Build') {
            agent{
                docker {
                    image 'node:18-alpine'
                    reuseNode true 
                }
            }
            steps {
                sh '''
                        ls -la 
                        node --version
                        npm --version
                        npm ci 
                        npm run build 
                        ls -la 
                '''
            }
        }
        stage ('Tests') {
            parallel {
                stage('Unit Tests') {
                    agent{
                        docker {
                            image 'node:18-alpine'
                            reuseNode true 
                        }
                    }
                    steps {
                    sh '''
                        echo 'Test stage'
                        test -f build/index.html
                        npm test
                    '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }
                stage('E2E') {
                    agent{
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true 
                        }
                    }
                    steps {
                        sh '''
                        npm install serve
                        node_modules/.bin/serve -s build &
                        npx playwright test
                        npx playwright test --reporter=line
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }
        stage('Deploy stage') {
            agent{
                docker {
                    image 'node:18-alpine'
                    reuseNode true 
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1 node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to stage. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
            }
        }
        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy prod') {
            agent{
                docker {
                    image 'node:18-alpine'
                    reuseNode true 
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                     node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent{
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                        reuseNode true 
                    }
                }
                    environment {
                        CI_ENVIRONMENT_URL='https://taupe-kulfi-4e08e0.netlify.app'
                }
                    steps {
                        sh '''
                        npx playwright test --reporter=line
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
    }
}
