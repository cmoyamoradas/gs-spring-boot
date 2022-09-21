pipeline {
    agent any
    environment {
        JURL = 'http://10.186.0.21'
        RT_URL = 'http://10.186.0.21/artifactory'
        TOKEN = credentials('token')
        ARTIFACTORY_LOCAL_DEV_REPO = 'demo-maven-dev-local'
        ARTIFACTORY_LOCAL_STAGING_REPO = 'demo-maven-staging-local'
        ARTIFACTORY_LOCAL_PROD_REPO = 'demo-maven-prod-local'
        CREDENTIALS = 'Artifactoryk8s'
        SERVER_ID = 'k8s'
        ARTIFACTORY_DOCKER_REGISTRY = '10.186.0.21/docker-local'
        DOCKER_REPOSITORY = 'docker-local'
        IMAGE_NAME = 'gs-spring-boot'
        IMAGE_VERSION = '1.0.0'
    }
    tools {
        maven "maven-3.6.3"
    }
    stages {
        stage ('Artifactory configuration') {
            steps {
                rtServer (
                    id: SERVER_ID,
                    url: RT_URL,
                    credentialsId: CREDENTIALS
                )
            }
        }
        stage ('Config JFrgo CLI') {
            steps {
                sh 'jf c add ${SERVER_ID} --interactive=false --overwrite=true --access-token=${TOKEN} --url=${JURL}'
                sh 'jf config use ${SERVER_ID}'
            }
        }
        stage ('Ping to Artifactory') {
            steps {
               sh 'jf rt ping'
            }
        }
        stage ('Config Maven'){
            steps {
                dir('complete'){
                    sh 'jf mvnc --repo-resolve-releases=demo-maven-virtual --repo-resolve-snapshots=demo-maven-virtual --repo-deploy-releases=demo-maven-virtual --repo-deploy-snapshots=demo-maven-virtual'
                }
            }
        }
        stage('Compile') {
            steps {
                echo 'Compiling'
                dir('complete') {
                    sh 'jf mvn clean test-compile -Dcheckstyle.skip -DskipTests'
                }
            }
        }
        stage('Package') {
            steps {
                dir('complete') {
                //Before creating the docker image, we need to create the .jar file
                    sh 'jf mvn package spring-boot:repackage -DskipTests -Dcheckstyle.skip'
                    echo 'Create the Docker image'
                    script {
                        docker.build(ARTIFACTORY_DOCKER_REGISTRY+'/'+IMAGE_NAME+':'+IMAGE_VERSION, '--build-arg JAR_FILE=target/*.jar .')
                    }
                }
            }
        }
        stage ('Push image to Artifactory') {
            steps {
                sh 'jf rt docker-push ${ARTIFACTORY_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_VERSION} ${DOCKER_REPOSITORY} --build-name="${JOB_NAME}" --build-number=${BUILD_ID} --url ${RT_URL} --access-token ${TOKEN}'
            }
        }
        stage ('Publish build info') {
            steps {
                // Collect environment variables for the build
                sh 'jf rt bce "${JOB_NAME}" ${BUILD_ID}'
                //Collect VCS details from git and add them to the build
                sh 'jf rt bag "${JOB_NAME}" ${BUILD_ID}'
                //Publish build info
                sh 'jf rt bp "${JOB_NAME}" ${BUILD_ID} --build-url=${BUILD_URL}'
                //Promote the build
                sh 'jf rt bpr --status=Development "${JOB_NAME}" ${BUILD_ID} ${DOCKER_REPOSITORY}'
                //Set properties to the files
                sh 'jf rt sp --include-dirs=true --build="${JOB_NAME}"/${BUILD_ID} "status=Development"'
            }
        }
        stage ('Approve Release for Staging') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                input message: "Are we good to go to Staging?"
            }
        }
        stage ('Release for Staging') {
            steps {
                sh 'jf rt bpr --status=Staging "${JOB_NAME}" ${BUILD_ID} ${DOCKER_REPOSITORY}'
                //Set properties to the files
                sh 'jf rt sp --build="${JOB_NAME}"/${BUILD_ID} "status=Staging"'
            }
        }
        stage ('Scan build') {
            steps {
                sh 'jf rt bs "${JOB_NAME}" ${BUILD_ID}'
            }
        }
       stage ('Approve Release for Production') {
           options {
               timeout(time: 5, unit: 'MINUTES')
           }
           steps {
               input message: "Are we good to go to Production?"
           }
       }
       stage ('Release for Production') {
           steps {
               sh 'jf rt bpr --status=Production "${JOB_NAME}" ${BUILD_ID} ${DOCKER_REPOSITORY}'
               //Set properties to the files
               sh 'jf rt sp --build="${JOB_NAME}"/${BUILD_ID} "status=Production"'
           }
       }
    }
}