#!groovy
pipeline {
    agent any
    environment {
        PROJECT_SCRATCH_PATH = "./config/project-scratch-def.json"
        SCRATCH_ORG_NAME     = "MyTestOrg"
        PERMISSION_SET       = "Geolocation"

        SERIAL  = System.currentTimeMillis()
        BRANCH  = env.BRANCH_NAME.replaceAll(/[\/\\]/, "")

        SFDX_HOME                      = tool 'toolbelt'
        SFDX_USE_GENERIC_UNIX_KEYCHAIN = true
        SFDX_AUTOUPDATE_DISABLE        = true
        HUB_ORG                        = "${HUB_ORG_DH}"
        SFDC_HOST                      = "${SFDC_HOST_DH}"
        JWT_KEY_CRED_ID                = "${JWT_CRED_ID_DH}"
        CONNECTED_APP_CONSUMER_KEY     = "${CONNECTED_APP_CONSUMER_KEY_DH}"
    }

    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        stage('Check SFDX functionality'){
            steps{
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: "jwt_key_file")]){
                    script{
                        sh "${SFDX_HOME}/sfdx --version"
                        sh "${SFDX_HOME}/sfdx force"
                    }
                }
            }
        }

        stage('Features RunLocalTests'){
            input {
                message "Should we run Local Tests?"
                ok "Yes, we should."
            }
            steps{
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: "jwt_key_file")]) {

                    script {
                        echo "on branch name: ${BRANCH}"
                        echo "1. DEV HUB auth"
                        def authStatus = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:auth:jwt:grant -i ${CONNECTED_APP_CONSUMER_KEY} -a DevHubOrg -u ${HUB_ORG} -f ${jwt_key_file} -s -r ${SFDC_HOST} --json --loglevel debug"
                        if (authStatus != 0) {
                            error "DEV HUB authorization failed"
                        } else {
                            echo "Successfully authorized to DEV HUB ${HUB_ORG}"
                            sh "${SFDX_HOME}/sfdx force:org:list --all"
                        }

                        echo "2. Creating Scratch Org"
                        SCRATCH_ORG_NAME="${SCRATCH_ORG_NAME}${BRANCH}" 
                        def orgStatus = sh returnStdout: true, script: "${SFDX_HOME}/sfdx force:org:create -s -f ${PROJECT_SCRATCH_PATH} -v ${HUB_ORG} -a ${SCRATCH_ORG_NAME} -d 1"
                        if (!orgStatus.contains("Successfully created scratch org")) {
                            error "Scratch Org creation failed"
                        } else {
                            echo orgStatus
                        }

                        echo "3. Pushing changes to Scratch Org"
                        def pushResult = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:source:push -u ${SCRATCH_ORG_NAME} -f"
                        if (pushResult != 0) {
                            error "Push failed"
                        } else {
                            echo "Metadata successfully pushed to the Org ${SCRATCH_ORG_NAME}"
                        }

                        def permissionResult = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:user:permset:assign -n ${PERMISSION_SET} -u ${SCRATCH_ORG_NAME}"
                        if (permissionResult != 0) {
                            error "Permission Set Assignment failed"
                        } else {
                            echo "Successfully assigned ${PERMISSION_SET}"
                        }

                        echo "4. Running all tests"
                        sh "mkdir -p tests/${SCRATCH_ORG_NAME}"
                        timeout(time: 10, unit: "MINUTES") {
                            def testsResult = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:apex:test:run -l RunLocalTests -d tests/${SCRATCH_ORG_NAME} -r tap -u ${SCRATCH_ORG_NAME}"
                            if (testsResult != 0) {
                                error "Apex tests run failed"
                            } else {
                                echo "Apex tests successfully run"
                            }
                        }

                        echo "5. Collecting tests result"
                        junit keepLongStdio: true, testResults: "tests/**/*-junit.xml"

                        echo "6. Clean up"
                        def deleteResult = sh returnStdout: true, script: "${SFDX_HOME}/sfdx force:org:delete -u ${SCRATCH_ORG_NAME} -p"
                        echo "Delete Org status " + deleteResult

                        echo "7. Check SFDX Limits"
                        sh "${SFDX_HOME}/sfdx force:limits:api:display -u DevHubOrg"
                    }
                }
            }
        }

        stage('MANUAL LOCAL TEST'){
            input {
                message "Should we run Local Tests?"
                ok "Yes, we should."
            }
            steps{
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: "jwt_key_file")]) {

                    script {
                        echo "on branch name: ${BRANCH}"
                        echo "1. DEV HUB auth"
                        def authStatus = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:auth:jwt:grant -i ${CONNECTED_APP_CONSUMER_KEY} -a DevHubOrg -u ${HUB_ORG} -f ${jwt_key_file} -s -r ${SFDC_HOST} --json --loglevel debug"
                        if (authStatus != 0) {
                            error "DEV HUB authorization failed"
                        } else {
                            echo "Successfully authorized to DEV HUB ${HUB_ORG}"
                            sh "${SFDX_HOME}/sfdx force:org:list --all"
                        }
                        /*
                        echo "2. Creating Scratch Org"
                        SCRATCH_ORG_NAME="${SCRATCH_ORG_NAME}${BRANCH}" 
                        def orgStatus = sh returnStdout: true, script: "${SFDX_HOME}/sfdx force:org:create -s -f ${PROJECT_SCRATCH_PATH} -v ${HUB_ORG} -a ${SCRATCH_ORG_NAME} -d 1"
                        if (!orgStatus.contains("Successfully created scratch org")) {
                            error "Scratch Org creation failed"
                        } else {
                            echo orgStatus
                        }
                        */
                        echo "3. Pushing changes to Scratch Org"
                        def pushResult = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:source:push -u QA_TestPR-7 -f"
                        if (pushResult != 0) {
                            error "Push failed"
                        } else {
                            echo "Metadata successfully pushed to the Org ${SCRATCH_ORG_NAME}"
                        }

                        def permissionResult = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:user:permset:assign -n ${PERMISSION_SET} -u QA_TestPR-7"
                        if (permissionResult != 0) {
                            error "Permission Set Assignment failed"
                        } else {
                            echo "Successfully assigned ${PERMISSION_SET}"
                        }

                        echo "4. Running all tests"
                        sh "mkdir -p tests/${SCRATCH_ORG_NAME}"
                        timeout(time: 10, unit: "MINUTES") {
                            def testsResult = sh returnStatus: true, script: "${SFDX_HOME}/sfdx force:apex:test:run -l RunLocalTests -d tests/${SCRATCH_ORG_NAME} -r tap -u QA_TestPR-7"
                            if (testsResult != 0) {
                                error "Apex tests run failed"
                            } else {
                                echo "Apex tests successfully run"
                            }
                        }

                        echo "5. Collecting tests result"
                        junit keepLongStdio: true, testResults: "tests/**/*-junit.xml"

                        echo "7. Check SFDX Limits"
                        sh "${SFDX_HOME}/sfdx force:limits:api:display -u DevHubOrg"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline successfully executed!"
        }
        failure {
            echo "Pipeline execution failed!"
        }
    }
}