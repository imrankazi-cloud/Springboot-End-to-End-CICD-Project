pipeline {
  agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
          }
    }
    stage('Build and Test') {
      steps {
        sh 'ls -ltr'
        // build the project and create a JAR file
        sh 'mvn clean package'
      }
    }

     stage('SonarQube Analysis') {
    environment {
        SONAR_AUTH_TOKEN = credentials('sonarqube')
        SONAR_URL = 'http://98.80.65.36:9000'  // Replace with actual URL
    }
    steps {
        sh '''
           mvn clean package org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3921:sonar \\
                -Dsonar.login=$SONAR_AUTH_TOKEN \\
                -Dsonar.host.url=$SONAR_URL
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
                        sh 'jfrog rt upload --url http://3.90.216.177:8082/artifactory/ --access-token ${ARTIFACTORY_ACCESS_TOKEN} target/*.jar springboot-web-app/'
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
