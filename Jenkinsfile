pipeline {
    agent any
    environment {
        JURL = 'http://jfrog.trickynickel.org'
        RT_URL = 'http://jfrog.trickynickel.org/artifactory'
        TOKEN = credentials('artifactory-gke-token')
        ARTIFACTORY_LOCAL_DEV_REPO = 'demo-maven-dev-local'
        SERVER_ID = 'gke'
        BUILD_NAME = "GS_SPRING_BOOT_main_mvn"
        BUILD_URL = "http://jenkins.trickynickel.org/job/${JOB_NAME}/${BUILD_ID}"
    }
    tools {
        maven "maven-3.6.3"
    }

    stages {
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
        stage ('Upload artifact') {
            steps {
                dir('complete') {
                    sh 'jf mvn clean deploy -Dcheckstyle.skip -DskipTests --build-name="${BUILD_NAME}" --build-number=${BUILD_ID}'
                }
            }
        }
        stage ('Publish build info') {
            steps {
                // Collect environment variables for the build
                sh 'jf rt bce "${BUILD_NAME}" ${BUILD_ID}'
                //Collect VCS details from git and add them to the build
                sh 'jf rt bag "${BUILD_NAME}" ${BUILD_ID}'
                //Publish build info
                sh 'jf rt bp "${BUILD_NAME}" ${BUILD_ID} --build-url=${BUILD_URL}'
                //Promote the build
                sh 'jf rt bpr --status=Development --props="status=Development" "${BUILD_NAME}" ${BUILD_ID} ${ARTIFACTORY_LOCAL_DEV_REPO}'
            }
        }
    }
}