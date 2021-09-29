pipeline {
    agent any

    options {
        timestamps() // Add timestamps to logging
        timeout(time: 8, unit: 'HOURS') // Abort pipleine

        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds()
    }
    environment {
        PATH = "/usr/local/bin:$PATH"
    }
    parameters {
        choice(name: 'TestName',
               choices: [
                   'Basic GoPath_TC',
                   'Allocation Mode_MMCP_TC',
                   'Allocation Required_MMCP_TC',
                   'Auto GPS-SDL_MMCP_TC',
                   'Block IV Non-GPS_MMCP_TC',
                   'Specific Tasks_MMCP_TC',
                   'Cancel_MM_TC',
                   'Interface VLS Maintain_MM_TC',
                   'SM3 Basic_MM_TC',
                   'ALL'
               ],
               description: 'Select a Testcase to run')
    }

    stages {
        stage('Init') {
            steps {
                echo "Stage: Init"
                echo "branch=${env.BRANCH_NAME}, test=${params.TestName}"
            }
        }
        stage('Common Config') {
            steps {
                echo 'Stage: Common Config'
                // Checkout repo with common config files/scripts to 'common' folder
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/main']],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'common']],
                          submoduleCfg: [], userRemoteConfigs: [[url: 'file:///home/jenkins/gitrepos/cicd-common']]
                         ])
            }
        }
        stage('Ping Servers') {
            steps {
                echo 'Stage: Ping Servers'
                dir('common') {
                    sh '''
                    ./pingAll.sh
                    '''
                }
            }
        }
        stage('Get Testcases') {
            options {
                timeout(time: 1, unit: 'MINUTES')
            }
            steps {
                echo 'Stage: Get Testcases'
                dir('common') {
                    sh '''
                    ./tm_get_testcases.sh
                    cat ./tomahawk-tpid.txt
                    cat ./tomahawk-tcid.txt
                    '''
                }
            }
        }
        stage('Test Manager') {
            steps {
                echo "Stage: Test Manager"
                dir('common') {
                    script {
                        echo "tcname: \'${params.TestName}\'"
                        def tcid = sh(script: "grep \"${params.TestName}\" tomahawk-tcid.txt | awk -F\':\' \'{print \$1}\'", returnStdout: true).trim()
                        echo "tcid: ${tcid}"
                        build(job: '/RunTestcaseId/main', parameters: [string(name: 'testcase_id', value: "${tcid}")], wait: true)
                    }
                }
            }
        }
        stage('Transfer Data') {
            steps {
                echo "Stage: Transfer Data"
                dir('common') {
                    // Transfer DX data from TTWCS to AM
                    sh '''
                    ./transfer_data_to_am.sh
                    '''
                }
            }
        }
        stage('Analysis Manager') {
            steps {
                echo "Stage: Analysis Manager"
                dir('common') {
                    script {
                        def idtag = sh(script: "cat currentDxFile", returnStdout: true).trim()
                        echo "Start Analysis Manager Job with idtag=${idtag}"
                        build(job: '/AnalysisMgr/main', parameters: [string(name: 'idtag', value: "${idtag}")], wait: true)
                    }
                }
            }
        }
        stage('Cleanup') {
            steps {
                echo "Stage: Cleanup"
            }
        }
    }
    post {
        always {
            echo "post/always"
            deleteDir()
        }
        success {
            echo "post/success"
        }
        failure {
            echo "post/failure"
        }
    }
}
