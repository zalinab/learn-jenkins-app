pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '1d9d533b-8b27-476c-bc64-edf9935bb765'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }
    stages {
        stage('Docker') {
            steps {
                sh 'docker build -t my-playwright .'
            }
        }
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
                            image 'my-playwright'
                            reuseNode true 
                        }
                    }
                    steps {
                        sh '''
                        node_modules/.bin/serve -s build &
                        npx playwright test
                        npx playwright test --reporter=line
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'localE2E.html', reportName: 'Local E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }
        stage('Deploy staging') {
            agent{
                docker {
                    image 'node:18-alpine'
                    reuseNode true 
                }
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to stage. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "node-jq -r '.deploy_url' deploy-output.json", returnStdout:true)
                    echo 'URL is saved'
                    echo env.STAGING_URL
                }
            }
        }
        stage('Staging E2E') {
            agent{
                docker {
                    image 'my-playwright'
                        reuseNode true 
                    }
                }
                    environment {
                        CI_ENVIRONMENT_URL="${STAGING_URL}"
                }
                    steps {
                        sh '''
                        npx playwright test --reporter=line
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'stageE2E.html', reportName: 'Playwright E2E Stage', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
        }
        stage('Deploy and E2E Prod') {
            agent{
                docker {
                    image 'my-playwright'
                    reuseNode true 
                    }
                }
                    environment {
                        CI_ENVIRONMENT_URL='https://taupe-kulfi-4e08e0.netlify.app'
                }
                    steps {
                        sh '''
                            node --version
                            netlify --version
                            echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                            netlify status
                            netlify deploy --dir=build --prod
                            npx playwright test --reporter=line
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'prodE2E.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
            }
        }
    }
}
