pipeline {
    agent {
        kubernetes {
            label 'molgenis'
        }
    }
    environment {
        SAUCELABS_CRED = credentials('molgenis-jenkins-saucelabs-secret')
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }
            }
        }
        stage('Build [ pull request ]') {
            when {
                changeRequest()
            }
            environment {
                //PR-1234-231
                TAG = "PR-${CHANGE_ID}-${BUILD_NUMBER}"
                //0.0.0-SNAPSHOT-PR-1234-231
                PREVIEW_VERSION = "0.0.0-SNAPSHOT-${TAG}"
            }
            steps {
                container('maven') {
                    sh "mvn versions:set -DnewVersion=${PREVIEW_VERSION} -DgenerateBackupPoms=false"
                    sh "mvn clean install -Dmaven.test.redirectTestOutputToFile=true -DskipITs -Ddockerfile.tag=${TAG} -Ddockerfile.skip=false"
                }
            }
        }

        stage('Build [ master ]') {
            when {
                branch 'master'
            }
            environment {
                TAG = 'latest'
            }
            steps {
                container('maven') {
                    sh "mvn clean install -Dmaven.test.redirectTestOutputToFile=true -DskipITs -Ddockerfile.tag=${TAG} -Ddockerfile.skip=false"
                }
            }
        }
        stage('Build [ x.x ]') {
            when {
                expression { BRANCH_NAME ==~ /[0-9]\.[0-9]/ }
            }
            environment {
                TAG = 'stable'
            }
            steps {
                container('maven') {
                    sh "mvn clean install -Dmaven.test.redirectTestOutputToFile=true -DskipITs -Ddockerfile.tag=${BRANCH_NAME}-${TAG} -Ddockerfile.skip=false"
                }
            }
        }
        stage('Release [ x.x ]') {
            when {
                expression { BRANCH_NAME ==~ /[0-9]\.[0-9]/ }
            }
            environment {
                TAG = 'stable'
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        env.RELEASE_SCOPE = input(
                                message: 'Do you want to release?',
                                ok: 'Release',
                                parameters: [
                                        choice(choices: 'candidate\nrelease', description: '', name: 'RELEASE_SCOPE')
                                ]
                        )
                    }
                }
                milestone 1
                container('maven') {
                    sh ".release/generate_release_properties.bash ${APP_NAME} ${ORG} ${env.RELEASE_SCOPE}"
                    sh "mvn release:prepare release:perform -Dmaven.test.redirectTestOutputToFile=true -DskipITs -Ddockerfile.tag=${BRANCH_NAME}-${TAG} -Ddockerfile.skip=false"
                }
            }
        }
    }
}
