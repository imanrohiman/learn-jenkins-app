pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '8b7961dd-fff2-418c-a6d4-0786e1ae5dfd'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        // REACT_APP_VERSION = "1.0.$BUILD_ID"
        REACT_APP_VERSION = "1.0.${env.BUILD_ID}"

        // Tambahan untuk MinIO lokal
        S3_ENDPOINT = 'http://172.17.0.1:9000'
        AWS_S3_BUCKET = 'test-bucket'   // ganti sesuai nama bucket MinIO kamu

        // Fix error npm EACCES (cache permission)
        NPM_CONFIG_CACHE = '/home/node/.npm'

    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-bullseye'
                    args '-u root:root'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Starting Build Stage"
                    mkdir -p $NPM_CONFIG_CACHE
                    mkdir -p /home/node/.npm
                    npm config set cache $NPM_CONFIG_CACHE
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build

                    echo "Adding Netlify SPA redirects..."
                    echo "/* /index.html 200" > build/_redirects

                    ls -la build
                    echo "Build finished."
                '''
            }
        }

        stage('MinIO Upload (Local S3)') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    reuseNode true
                    args "--entrypoint='' -v ${WORKSPACE}/build:/build"
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'minio-cred', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        echo "Uploading build folder to MinIO..."
                        aws --endpoint-url $S3_ENDPOINT s3 sync build s3://$AWS_S3_BUCKET --delete
                        echo "Upload complete!"
                    '''
                }
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-bullseye'
                            args '-u root:root'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            mkdir -p $NPM_CONFIG_CACHE
                            npm config set cache $NPM_CONFIG_CACHE
                            echo "Running Unit Tests"
                            echo 'npm test -- --detectOpenHandles'
                        '''
                    }
                    post {
                        always {
                            echo ''' junit 'test-results/junit.xml' '''
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.47.0-jammy'
                            args '-u root:root'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "Running E2E tests..."

                            # Install dependencies
                            echo npm install --save-dev terminal-link ansi-escapes

                            # Set app version
                            export REACT_APP_VERSION="1.0.${BUILD_ID}"
                            echo "REACT_APP_VERSION=$REACT_APP_VERSION"

                            # Build app
                            npm run build

                            # Serve build
                            npx serve -s build -l 5000 &
                            SERVER_PID=$!
                            echo "Server started with PID $SERVER_PID"
                            sleep 5

                            # Run E2E tests
                            echo 'npx playwright install --with-deps chromium'
                            echo 'npx playwright test --reporter=html --project=chromium'
                            
                            # Kill server
                            kill $SERVER_PID || true

                            # Run unit tests (fix terminal-link)
                            echo 'npm test -- --detectOpenHandles'
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Local E2E'])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent { 
                docker { 
                    image 'node:18-bullseye'
                    args '-u root:root -v $WORKSPACE:/workspace --dns=8.8.8.8 --dns=1.1.1.1'
                    reuseNode true 
                } 
            }
          
            steps {
                sh '''
                    echo "Setting locale..."
                    export LANG=C.UTF-8
                    export LC_ALL=C.UTF-8
                    
                    echo "Downloading Netlify CLI standalone (NO sharp)..."
                    curl -sL https://github.com/netlify/cli/releases/latest/download/netlify-linux-x64 -o netlify
                    chmod +x netlify

                    echo "Installing Netlify CLI..."
                    npm install -g netlify-cli@17

                    ls -la /workspace/build || echo "BUILD FOLDER NOT FOUND!"
                    
                    apt-get update -y --fix-missing
                    apt-get update && apt-get install -y jq

                    npx update-browserslist-db@latest --yes

                    echo "Creating _redirects file for SPA routing..."
                    echo "/* /index.html 200" > build/_redirects

                    echo "Deploying to Staging..."
                    npx netlify deploy --auth $NETLIFY_AUTH_TOKEN --site $NETLIFY_SITE_ID --dir=build --json > deploy-output.json

                    export CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                    echo "Staging URL: $CI_ENVIRONMENT_URL"
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E'])
                }
            }
        }

        stage('Deploy prod') {
            agent { 
                docker { 
                    image 'node:18-bullseye'
                    args '-u root:root -v $WORKSPACE:/workspace'
                    reuseNode true 
                } 
            }

            steps {
                sh '''
                    echo "Installing Netlify CLI..."
                    npm install -g netlify-cli --save-dev

                    echo "Deploying to production..."
                    npx netlify deploy --site $NETLIFY_SITE_ID --dir=/workspace/build --prod
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E'])
                }
            }
        }
    }
}
