pipeline {
    agent any
    environment {
        JURL = 'http://10.186.0.21'
        RT_URL = 'http://10.186.0.21/artifactory'
        TOKEN = 'eyJ2ZXIiOiIyIiwidHlwIjoiSldUIiwiYWxnIjoiUlMyNTYiLCJraWQiOiJoMWQ5eV91V2tfVGFrY1ZlZ3c5ZG5sM2xFSWZObFI3cDdGckN5aHRHS3kwIn0.eyJleHQiOiJ7XCJyZXZvY2FibGVcIjpcInRydWVcIn0iLCJzdWIiOiJqZmFjQDAxZzdwcjAzaG5xbWN0MXoxeG4xYWgxYjR6XC91c2Vyc1wvYWRtaW4iLCJzY3AiOiJhcHBsaWVkLXBlcm1pc3Npb25zXC9hZG1pbiIsImF1ZCI6IipAKiIsImlzcyI6ImpmZmVAMDAwIiwiZXhwIjoxNjg5NTQ2ODE5LCJpYXQiOjE2NTgwMTA4MTksImp0aSI6ImY2ZmE0ZDE5LTg1NzMtNDU0Zi05OGM1LWVkOWI5NWIxODZlYyJ9.J6SxPbF6KB-ocyuFmqrcKmsrcrdm8yCuKzTsI0c5vMHd7u0ju_gSpY3MHXz1fJreS9wQEVI0MIoR3fSoOLZyMTYAFiDV3RboQ9AdsVb2MQfOiPIv32MwqUw3TCO3zAwZv7TteGhj3amfKn96rJTtFw2PrXVLZh3mtWQoSanvsrc1O4wcmF0pm5169GlXd0LRldcZnv6ItrsLbxXMO6Tpy7apOIdNBlg3VBWE-AUQMzneRlm2f9Uxo42ldBNXNKDzmidE1-PT37rfaYpHj698ja7OWCtzK6kV5V8AsF2TPxOym1bRh0oNWDu4lP-pY4fRSIgfNFPzn43M693r02jwwA'
        ARTIFACTORY_LOCAL_DEV_REPO = 'demo-maven-dev-local'
        ARTIFACTORY_LOCAL_STAGING_REPO = 'demo-maven-staging-local'
        ARTIFACTORY_LOCAL_PROD_REPO = 'demo-maven-prod-local'
        CREDENTIALS = 'Artifactoryk8s'
        SERVER_ID = 'k8s'
        promotion = true
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
        stage ('Upload artifact') {
            steps {
                dir('complete') {
                    sh 'jf mvn clean deploy -Dcheckstyle.skip -DskipTests --build-name="${JOB_NAME}" --build-number=${BUILD_ID}'
                }
            }
        }
        stage ('Publish build info') {
            steps {
                // Collect environment variables for the build
                sh 'jf rt bce --build-name="${JOB_NAME}" --build-number=${BUILD_ID}'
                //Collect VCS details from git and add them to the build
                sh 'jf rt bag --build-name="${JOB_NAME}" --build-number=${BUILD_ID}'
                //Publish build info
                sh 'jf rt bp --build-name="${JOB_NAME}" --build-number=${BUILD_ID}'
            }
        }
        /*
        stage ('Publish build info') {
            steps {
                rtPublishBuildInfo (
                    buildName: JOB_NAME,
                    buildNumber: BUILD_ID,
                    serverId: SERVER_ID
                )
            }
        }
        */
        stage ('Approve Release for Staging') {
            steps {
                input message: "Are we good to go to Staging?"
            }
        }
        stage ('Release for Staging') {
            steps {
                println "Release for Staging approved"
                rtPromote (
                    //Mandatory parameter
                    serverId: SERVER_ID,
                    targetRepo: ARTIFACTORY_LOCAL_STAGING_REPO,
                    //Optional parameters
                    buildName: JOB_NAME,
                    buildNumber: BUILD_ID,
                    comment: 'We are good for Staging',
                    sourceRepo: ARTIFACTORY_LOCAL_DEV_REPO,
                    status: 'Staging',
                    includeDependencies: true,
                    failFast: true
                    //copy: true
                )
            }
        }
       stage ('Approve Release for Production') {
           steps {
               input message: "Are we good to go to Production?"
           }
       }
       stage ('Release for Production') {
           steps {
               println "Release for Production approved"
               rtPromote (
                   //Mandatory parameter
                   serverId: SERVER_ID,
                   targetRepo: ARTIFACTORY_LOCAL_PROD_REPO,
                   //Optional parameters
                   buildName: JOB_NAME,
                   buildNumber: BUILD_ID,
                   comment: 'We are good to go to Production',
                   sourceRepo: ARTIFACTORY_LOCAL_STAGING_REPO,
                   status: 'Production',
                   includeDependencies: true,
                   failFast: true
                   //copy: true
               )
           }
       }
    }
}