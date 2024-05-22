@Library("shared-library") _

pipeline {
    environment {
        AWS_REGION                        = credentials('ds-aws-region')
        FILE_NAME2                        = './Python-Code/RunConfiguration.ini'
        SIMULATED_USERS                   = sh(script: "echo `awk -F 'SIMULATED_USERS:' '{print \$2}' $FILE_NAME2`", returnStdout: true).trim()
        HOLD_TIME                         = sh(script: "echo `awk -F 'HOLD_TIME:' '{print \$2}' $FILE_NAME2`", returnStdout: true).trim()
        USERS                             = sh(script: "echo `awk -F 'NO_OF_USERS:' '{print \$2}' $FILE_NAME2`", returnStdout: true).trim() 
        TESTCASE                          = sh(script: "echo `awk -F 'TEST_FILE_NAME:' '{print \$2}' $FILE_NAME2`", returnStdout: true).trim()
        GITHUB_TOKEN                      = credentials('ds-github-cred')
        GH_TOKEN                          = "${GITHUB_TOKEN_PSW}"
        BUILD_TIME                        = sh(script: "echo `date +%F-%H-%M-%S`", returnStdout: true).trim()
        GITHUB_URL                        = "https://github.com/demandscience/lastbounce-ui-plt"
        GIT_REPO                          = "lastbounce-ui-plt"
    }

    options {
        ansiColor("xterm")
        timeout(time: pipelineTimeout(), unit: 'HOURS')
    }

    agent{
        label "spot-instance"
    }

    stages {

        // PR Information
        stage ('GET PR INFO') {
            when {
                allOf {
                    not { expression { BRANCH_NAME ==~ /(PR-.*)/} }
                }
            }
            environment {
                OPEN_PR   = "OPEN_PR.json"
                MERGED_PR = "MERGED_PR.json"
            }
            steps {
                script {
                    if(env.GIT_BRANCH ==~ /(DEVELOP|STAGING|PRE-PROD|PROD)/){
                        sh "gh pr list -B ${GIT_BRANCH} -L 1 --state merged > ${OPEN_PR}"
                        def PR_NUM =  sh(script: "awk '{print \$1}' $OPEN_PR", returnStdout: true).trim()
                        sh "gh pr view ${PR_NUM} > ${MERGED_PR}"
                        sh "cat ${MERGED_PR}"
                    }
                    else{
                        sh "gh pr view ${GIT_BRANCH} > ${OPEN_PR}"
                        sh "cat ${OPEN_PR}"
                    }
                }
            }
        }

        // Smoke Test
        stage('Install Jmeter') {
            steps {
                script {
                    sh "wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.5.tgz >/dev/null"
                    sh "sudo tar -xvzf apache-jmeter-5.5.tgz >/dev/null"
                    sh "mkdir jmeter && sudo mv apache-jmeter-5.5/* jmeter/ && rm -rf apache-jmeter-5.5.tgz apache-jmeter-5.5"
                    sh "sudo chmod -R 777 jmeter/bin/user.properties"
                    sh "sudo echo 'jmeter.save.saveservice.output_format=csv' >> jmeter/bin/user.properties"
                    sh "cd jmeter/lib && sudo wget https://repo1.maven.org/maven2/kg/apc/cmdrunner/2.3/cmdrunner-2.3.jar"
                    sh "cd jmeter/lib/ext && sudo wget https://repo1.maven.org/maven2/kg/apc/jmeter-plugins-manager/1.8/jmeter-plugins-manager-1.8.jar"
                    sh "cd jmeter/lib && sudo java -jar cmdrunner-2.3.jar --tool org.jmeterplugins.repository.PluginManagerCMD install jpgc-casutg"
                }
            }
        }

        
        // Regression Test
        stage('Performance Load Test') {
            when {
                expression { BRANCH_NAME ==~ /(DEVELOP|STAGING|PRE-PROD|PROD)/}
            }
            steps {
                script {
                    switch_branches()
                    echo "Testing Jmeter PLT"
                    sh "export HEAP='-Xms2048m -Xmx2048m -X:MaxMetaspaceSize=2048m'"
                    sh "sudo jmeter/bin/jmeter.sh -n -t TestScenarios/${env.TESTCASE} -l jmeter/bin/result.csv -j jmeter/bin/logfile.log -i log4j2.xml -e -o jmeter/html-reports -Jhost=${HOST} -Jusers=${env.USERS} -Jsim_users=${env.SIMULATED_USERS} -Jhold_time=${env.HOLD_TIME}"
                }
            }
            post {
                success {
                    script {
                        publishHTML target: [
                            allowMissing: false,
                            alwaysLinkToLastBuild: false,
                            keepAll: true,
                            reportDir: 'jmeter/html-reports',
                            reportFiles: '*.html',
                            reportName: 'Performance Test Report',
                            reportTitles: 'Performance Test Report'
                        ]
                        archiveArtifacts artifacts: 'jmeter/bin/logfile.log', fingerprint: true
                        archiveArtifacts artifacts: 'jmeter/bin/result.csv', fingerprint: true
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                if (currentBuild.rawBuild.getCause(hudson.triggers.TimerTrigger$TimerTriggerCause) == null && currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause) == null && GIT_BRANCH ==~ /(DEVELOP|STAGING|PRE-PROD|PROD)/) {
                    withCredentials([gitUsernamePassword(credentialsId: 'ds-github-cred')]) {
                        sh "git tag ${GIT_REPO}-${GIT_BRANCH}-${BUILD_TIME}-${BUILD_NUMBER}"
                        sh "git push ${GITHUB_URL}.git  ${GIT_REPO}-${GIT_BRANCH}-${BUILD_TIME}-${BUILD_NUMBER}"
                    }
                }
            }
            
        }
        failure {
            script {
                slackNotification.sendFailureSlackMsg(slackNotification.getSlackChannel())
                cleanWs()
            }
        }
        aborted {
            script {
                def currentResult = currentBuild.currentResult
                echo "${currentResult}"
                if (currentResult == 'ABORTED') {
                    if (checkTimeoutExceed.isTimeoutExceeded()) {
                        sendEmailNotification.pipelineTimeoutEmail()
                    }
                    else {
                        echo "Don't send any email"
                    }
                }
            }
        }
    }
}

void switch_branches() {
    switch(GIT_BRANCH) {
        case "DEVELOP":
            
            HOST        = "app.lastbounce-develop.demandscience-apps.com"
            break
        case "STAGING":
            
            HOST        = "app.lastbounce-develop.demandscience-apps.com"
            break
        case "PRE-PROD":
            
            HOST        = "app.lastbounce-develop.demandscience-apps.com"
            break
        case "PROD":
           
            HOST        = "app.lastbounce-develop.demandscience-apps.com"
            break
        default:
            break
    }
}


