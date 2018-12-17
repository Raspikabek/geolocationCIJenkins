#!groovy
pipeline {

    //execute this Pipeline, or stage, on any available agent.
    agent any

    //specifying variables for all stages
    environment {
        //project vars
        PROJECT_SCRATCH_PATH = "./config/project-scratch-def.json"
        SCRATCH_ORG_NAME     = "Test"
        PERMISSION_SET       = "Geolocation"
        CC_4_6_ID            = "04t0V000001Dyaf"
        USER_NAME            = "User"

        //Jenkins env vars
        SERIAL  = System.currentTimeMillis()
        BRANCH  = env.BRANCH_NAME.replaceAll(/[\/\\]/, "")

        //SFDX vars
        SFDX_HOME                      = tool 'toolbelt'
        SFDX_USE_GENERIC_UNIX_KEYCHAIN = true
        SFDX_AUTOUPDATE_DISABLE        = true
        HUB_ORG                        = "${HUB_ORG_DH}"
        SFDC_HOST                      = "${SFDC_HOST_DH}"
        JWT_KEY_CRED_ID                = "${JWT_CRED_ID_DH}"
        CONNECTED_APP_CONSUMER_KEY     = "${CONNECTED_APP_CONSUMER_KEY_DH}"
    }

    //specifying global execution timeout of one hour, after which Jenkins will abort the Pipeline run.
    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {

        /*stage("Checkout source") {
            steps {
                checkout scm
            }
        }*/

        stage('Check SFDX functionality'){
            steps{
                withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: "jwt_key_file")]){
                    script{
                        sh "${SFDX_HOME}/sfdx --version"
                        sh "${SFDX_HOME}/sfdx force"
                        /*def myRandomUserName = sh returnStdout: true, script: "cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1 | tr -d '\n'"
                        tempUserName="qaUser-${myRandomUserName}-${env.BUILD_ID}@scratch-pexlify.com"
                        echo "${tempUserName}"*/
                    }
                }
            }
        }

        stage('Features RunLocalTests'){
            when{
                beforeInput true
                branch 'feature/*'
            }
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
                            sh "${SFDX_HOME}/sfdx force:org:list --all"
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

        stage('PR-merge build'){
            when{
                beforeInput true
                branch 'PR-*'
            }
            stages{
                stage('Create QA Environment'){
                    input {
                        message "Should we continue with QA?"
                        ok "Yes, we should."
                    }
                    steps {
                        echo 'Creating QA Scratch Org...'
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
                                SCRATCH_ORG_NAME="QA_${SCRATCH_ORG_NAME}${BRANCH}" 
                                def orgStatus = sh returnStdout: true, script: "${SFDX_HOME}/sfdx force:org:create -s -f ${PROJECT_SCRATCH_PATH} -v ${HUB_ORG} -a ${SCRATCH_ORG_NAME} -d 1"
                                if (!orgStatus.contains("Successfully created scratch org")) {
                                    error "Scratch Org creation failed"
                                } else {
                                    echo orgStatus
                                    sh "${SFDX_HOME}/sfdx force:org:list --all"
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

                                echo "6. Create User"
                                def myRandomUserName = sh returnStdout: true, script: "cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1 | tr -d '\n'"
                                tempUserName="qaUser-${myRandomUserName}-${env.BUILD_ID}@scratch-pexlify.com"

                                def userCreationResult = sh returnStdout: true, script: "${SFDX_HOME}/sfdx force:user:create -a qaUser -f config/user-def.json City=Dublin UserName=${tempUserName}"
                                echo "User Creation status: " + userCreationResult
                                sh "${SFDX_HOME}/sfdx force:user:display -u qaUser"

                                echo "7. Check SFDX Limits"
                                sh "${SFDX_HOME}/sfdx force:limits:api:display -u DevHubOrg"
                            }
                        } 
                    }
                }
                stage('Create UAT Environment'){
                    input {
                        message "Should we continue with UAT?"
                        ok "Yes, we should."
                    }
                    steps {
                        echo 'Creating UAT Scratch Org... (no flow)'
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