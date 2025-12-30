pipeline {
    parameters {
        // Checkbox Parameter, If checked Artifact will upload otherwise skip the stage
        booleanParam(
            defaultValue: false,
            description: 'Upload Artifact?',
            name: 'YES'
        )
        options {
        skipDefaultCheckout()
        timeout(time: 30, unit: 'MINUTES')
    }
    }
    agent {
        // Docker Image where Maven and Docker Installed already, So don't need to configure separately
        docker {
            image 'abhishekf5/maven-abhishek-docker-agent:v1'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    stages {
         stage('Clean Workspace') {
            steps {
                script {
                    // Force delete everything
                    sh '''
                        rm -rf target .git/logs .git/refs/remotes/origin
                        git clean -fdx
                    '''
                }
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        // Building the Maven Project
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Analysis') {
    environment {
        SONAR_AUTH_TOKEN = credentials('sonarqube')
        SONAR_HOST_URL = 'http://52.90.186.100:9000/'  // Update URL
    }
    steps {
        sh '''
            mvn clean package org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3921:sonar \
            -Dsonar.login=$SONAR_AUTH_TOKEN \
            -Dsonar.host.url=$SONAR_HOST_URL
        '''
            }
        }


        // Uploading code Artifact to the JFrog Artifactory
        stage('Upload Code Artifacts') {
            agent {
                // Docker Image for JFrog-CLI
                docker {
                    image 'releases-docker.jfrog.io/jfrog/jfrog-cli-v2:2.2.0'
                    reuseNode true
                }
            }
            // credentials will take from the Jenkins Environment Credentials
            environment {
                CI = true
                ARTIFACTORY_ACCESS_TOKEN = credentials('artifactory-access-token')
            }
            steps {
                script {
                    // If Checkbox tick then, Perform this stage
                    if (params.YES) {
                        sh 'jfrog rt upload --url http://54.234.32.232:8082/artifactory/ --access-token ${ARTIFACTORY_ACCESS_TOKEN} target/*.jar springboot-web-app/'
                    } else {
                        // If Checkbox not tick then, Skip this stage and go for the next stage
                        return
                    }
                }
            }
        }
        // Building the Image and Pushing it to DockerHub
        stage('Build & Push Docker Image') {
            // Credentials of DockerHub which stored in the Jenkins Credentials
            environment {
                DOCKER_IMAGE = "imranawsdevops/spring-docker:${BUILD_NUMBER}"
                REGISTRY_CREDENTIALS = credentials('docker-cred')
            }
            steps {
                // Building the Docker Image and Pushing it... After Pushing, make sure remove the Image because the it will take more storage
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                    def dockerImage = docker.image("${DOCKER_IMAGE}")
                    docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                        dockerImage.push()
                    }
                    sh 'docker rmi ${DOCKER_IMAGE}'
                }
            }
        }
        // Updating Deployment file for BUILD_NUMBER 
        stage('Updating Deployment File') {
            // GIT Repo and username
            environment {
                GIT_REPO_NAME = "Springboot-End-to-End-CICD-Project"
                GIT_USER_NAME = "imrankazi-cloud"
            }
            steps {
                // Replacing the previous BUILD_NUMBER with NEW_BUILD_NUMBER and pushing the changes to Github
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "imrankaws@gmail.com"
                        git config user.name "imrankazi-cloud"
                        BUILD_NUMBER=${BUILD_NUMBER}
                        imageTag=$(grep -oP '(?<=spring-docker:)[^ ]+' deployment.yml)
                        sed -i "s/spring-docker:${imageTag}/spring-docker:${BUILD_NUMBER}/" deployment.yml
                        git add deployment.yml
                        git commit -m "Update deployment Image to version \${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:master
                    '''
                }
            }
        }
    }
}
