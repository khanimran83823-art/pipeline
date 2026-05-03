# myntra pipeline 
---
pipeline {
    agent any

tools {
    jdk 'jdk17'
    nodejs 'node16'
}

environment {
    SCANNER_HOME          = tool 'sonar-scanner'
    DOCKER_IMAGE          = 'myntra'
    DOCKER_REGISTRY       = 'khanimran83823'
    DOCKER_CREDENTIALS_ID = 'docker-cred'
    MANIFEST_FILE         = 'k8s/deployment.yml'
    GIT_REPO_NAME         = 'Project-Myntra-Clone'
    GIT_USER_NAME         = 'khanimran83823-art'
    GIT_EMAIL             = 'khanimran83823@gmail.com'
}

stages {
    stage('Clean Workspace') {
        steps {
            cleanWs()
        }
    }

    stage('Checkout Code') {
        steps {
            git branch: 'main', url: "https://github.com/${env.GIT_USER_NAME}/${env.GIT_REPO_NAME}.git"
        }
    }

    stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('sonar-server') {
                sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=Myntra \
                    -Dsonar.projectKey=Myntra
                """
            }
        }
    }

    stage('Quality Gate') {
        steps {
            waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
        }
    }

    stage('Install Dependencies') {
        steps {
            sh 'npm install'
        }
    }

    stage('Build & Push Docker Image') {
        steps {
            script {
                def imageTag = "${DOCKER_IMAGE}:${BUILD_NUMBER}"
                def registryImageTag = "${DOCKER_REGISTRY}/${imageTag}"
                
                sh "export NODE_OPTIONS='--max-old-space-size=2048' && docker build -t ${imageTag} ."

                withDockerRegistry(credentialsId: DOCKER_CREDENTIALS_ID, toolName: 'docker') {
                    sh """
                        docker tag ${imageTag} ${registryImageTag}
                        docker push ${registryImageTag}
                    """
                }
            }
        }
    }

    stage('Update Manifest and Push to GitHub') {
        steps {
            script {
                def newImage = "${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${BUILD_NUMBER}"
                withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh """
                        git config user.email "${GIT_EMAIL}"
                        git config user.name "${GIT_USER_NAME}"
                        sed -i 's|image: .*|image: ${newImage}|g' ${MANIFEST_FILE}
                        git add ${MANIFEST_FILE}
                        git commit -m "Update image to ${BUILD_NUMBER}" || echo "No changes"
                        git push https://${GIT_USER}:${GIT_PASS}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
                    """
                }
            }
        }
    }
}
---
