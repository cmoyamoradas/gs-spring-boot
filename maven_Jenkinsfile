pipeline {
    agent none
    environment {
        JURL = 'http://10.186.0.21'
        RT_URL = 'http://10.186.0.21/artifactory'
        TOKEN = 'eyJ2ZXIiOiIyIiwidHlwIjoiSldUIiwiYWxnIjoiUlMyNTYiLCJraWQiOiJoMWQ5eV91V2tfVGFrY1ZlZ3c5ZG5sM2xFSWZObFI3cDdGckN5aHRHS3kwIn0.eyJleHQiOiJ7XCJyZXZvY2FibGVcIjpcInRydWVcIn0iLCJzdWIiOiJqZmFjQDAxZzdwcjAzaG5xbWN0MXoxeG4xYWgxYjR6XC91c2Vyc1wvYWRtaW4iLCJzY3AiOiJhcHBsaWVkLXBlcm1pc3Npb25zXC9hZG1pbiIsImF1ZCI6IipAKiIsImlzcyI6ImpmZmVAMDAwIiwiZXhwIjoxNjg5NTQ2ODE5LCJpYXQiOjE2NTgwMTA4MTksImp0aSI6ImY2ZmE0ZDE5LTg1NzMtNDU0Zi05OGM1LWVkOWI5NWIxODZlYyJ9.J6SxPbF6KB-ocyuFmqrcKmsrcrdm8yCuKzTsI0c5vMHd7u0ju_gSpY3MHXz1fJreS9wQEVI0MIoR3fSoOLZyMTYAFiDV3RboQ9AdsVb2MQfOiPIv32MwqUw3TCO3zAwZv7TteGhj3amfKn96rJTtFw2PrXVLZh3mtWQoSanvsrc1O4wcmF0pm5169GlXd0LRldcZnv6ItrsLbxXMO6Tpy7apOIdNBlg3VBWE-AUQMzneRlm2f9Uxo42ldBNXNKDzmidE1-PT37rfaYpHj698ja7OWCtzK6kV5V8AsF2TPxOym1bRh0oNWDu4lP-pY4fRSIgfNFPzn43M693r02jwwA'
        ARTIFACTORY_LOCAL_DEV_REPO = 'demo-maven-dev-local'
        ARTIFACTORY_LOCAL_STAGING_REPO = 'demo-maven-staging-local'
        ARTIFACTORY_LOCAL_PROD_REPO = 'demo-maven-prod-local'
        SERVER_ID = 'k8s'
    }
    tools {
        maven "maven-3.6.3"
    }

    stages {
        stage ('Config JFrgo CLI') {
            agent any
            steps {
                sh 'jf c add ${SERVER_ID} --interactive=false --overwrite=true --access-token=${TOKEN} --url=${JURL}'
                sh 'jf config use ${SERVER_ID}'
            }
        }
        stage ('Ping to Artifactory') {
            agent any
            steps {
               sh 'jf rt ping'
            }
        }
        stage ('Config Maven'){
            agent any
            steps {
                dir('complete'){
                    sh 'jf mvnc --repo-resolve-releases=demo-maven-virtual --repo-resolve-snapshots=demo-maven-virtual --repo-deploy-releases=demo-maven-virtual --repo-deploy-snapshots=demo-maven-virtual'
                }
            }
        }
        stage('Compile') {
            agent any
            steps {
                echo 'Compiling'
                dir('complete') {
                    sh 'jf mvn clean test-compile -Dcheckstyle.skip -DskipTests'
                }
            }
        }
        stage ('Upload artifact') {
            agent any
            steps {
                dir('complete') {
                    sh 'jf mvn clean deploy -Dcheckstyle.skip -DskipTests --build-name="${JOB_NAME}" --build-number=${BUILD_ID}'
                }
            }
        }
        stage ('Publish build info') {
            agent any
            steps {
                // Collect environment variables for the build
                sh 'jf rt bce "${JOB_NAME}" ${BUILD_ID}'
                //Collect VCS details from git and add them to the build
                sh 'jf rt bag "${JOB_NAME}" ${BUILD_ID}'
                //Publish build info
                sh 'jf rt bp "${JOB_NAME}" ${BUILD_ID} --build-url=${BUILD_URL}'
                //Promote the build
                sh 'jf rt bpr --status=Development "${JOB_NAME}" ${BUILD_ID} ${ARTIFACTORY_LOCAL_DEV_REPO}'
                //Set properties to the files
                sh 'jf rt sp --build="${JOB_NAME}"/${BUILD_ID} "status=Development"'
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
            agent any
            steps {
                sh 'jf rt bpr --source-repo=${ARTIFACTORY_LOCAL_DEV_REPO} --status=Staging "${JOB_NAME}" ${BUILD_ID} ${ARTIFACTORY_LOCAL_STAGING_REPO}'
                //Set properties to the files
                sh 'jf rt sp --build="${JOB_NAME}"/${BUILD_ID} "status=Staging"'
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
           agent any
           steps {
               sh 'jf rt bpr --source-repo=${ARTIFACTORY_LOCAL_STAGING_REPO} --status=Production "${JOB_NAME}" ${BUILD_ID} ${ARTIFACTORY_LOCAL_PROD_REPO}'
               //Set properties to the files
               sh 'jf rt sp --build="${JOB_NAME}"/${BUILD_ID} "status=Production"'
           }
       }
    }
}