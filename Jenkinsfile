pipeline {
    agent none // The pipeline does not have a default agent; each stage specifies its own agent.
    
    environment{
        ServerName = 'ubuntu-AppServer-3120' // Server label
        DockerhubCredentials = 'dockerhub_credentials' // DockerHub credentials ID
        SnykToken = 'synk_api' // Snyk API token
        GithubRepo = "penjack/snaketest" // Name of the Github repository
        SONAR_PROJECT_KEY = 'snaketest'
    }

    stages {
        stage('CLONING GIT REPOSITORY') {
            // Cloning the files from github to the Appserver.
            agent {
                // Specifies which server this stage is run on.
                label ServerName
            }
            steps {
                // Checkout the source code from the Git repository defined in Jenkins
                checkout scm
            }
        }  

        stage('SECURITY TESTING'){
            parallel{
                stage('STATIC SECURITY TESTING WITH SYNK') {
                    // Static testing of the third party code from github.
                    agent { label ServerName }
                    steps {
                        script {
                            // Perform static analysis of the project code using Snyk
                            snykSecurity(
                                snykInstallation: 'Snyk', // Reference to the configured Snyk installation in Jenkins.
                                snykTokenId: SnykToken, // Use the provided Snyk API token.
                                severity: 'critical' // Set the severity level for not allowing code to continue.
                            )
                        }
                    }
                }

                stage('DYNAMIC SECURITY TESTING WITH SONARQUBE') {
                    // Dynamic testing of the Developer's code with SonarQube.
                    agent { label ServerName }
                    steps {
                        script {
                            // Perform dynamic code analysis using SonarQube
                            def scannerHome = tool name: 'SonarQubeScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'                     
                            withSonarQubeEnv('sonarqube') { // Use the SonarQube environment defined in Jenkins
                                sh """
                                ${scannerHome}/bin/sonar-scanner \
                                    -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                    -Dsonar.sources=.
                                """
                            }
                        }
                    }
                }
            }      
        }
        

        stage('BUILD AND TAG IMAGE') {
            // Build Docker image with the tag "Latest".
            agent { label ServerName }
            steps {
                script {
                    // Build a Docker image for the application
                    def app = docker.build(GithubRepo) // Build the image with the specified name
                    app.tag("latest") // Tag the image with 'latest'
                }
            }
        }

        stage('POST IMAGE TO DOCKERHUB') {    
            // Push the new Docker image to DockerHub with the tag Latest.
            agent { label ServerName }
            steps {
                script {
                    // Authenticate with DockerHub and push the built Docker image
                    docker.withRegistry('https://registry.hub.docker.com', DockerhubCredentials) {
                        def app = docker.image(GithubRepo) // Reference the Docker image
                        app.push("latest") // Push the image with the 'latest' tag
                    }
                }
            }
        }

        stage('PREPARE ENVIRONMENT') {
            agent { label ServerName }
            steps {
                script {
                    // Find and stop any Docker container using port 80 so there isn't any conflicts when deploying.
                        sh '''
                        CONTAINER_ID=$(docker ps -q --filter "publish=80")
                        if [ -n "$CONTAINER_ID" ]; then
                            echo "Stopping container using port 80: $CONTAINER_ID"
                            docker stop $CONTAINER_ID
                            docker rm $CONTAINER_ID
                        else
                            echo "No container using port 80."
                        fi
                        '''
                }
            }
        }

        stage('DEPLOYMENT') {    
            // Deploy the Image.
            agent { label ServerName }
            steps {
                // Deploy the application using Docker Compose
                sh "docker-compose down"    // Stop any existing containers defined in the Docker Compose file
                sh "docker-compose up -d"   // Start the containers in detached mode
            }
        }
    }
}
