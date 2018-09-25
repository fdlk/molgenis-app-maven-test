pipeline {
    agent {
        kubernetes {
            label 'molgenis'
        }
    }
    stages {
        stage('Retrieve build secrets') {
            steps {
                container('vault') {
                    script {
                        env.GITHUB_USER = sh(script: 'vault read -field=username secret/ops/token/github', returnStdout: true)
                        env.GITHUB_TOKEN = sh(script: 'vault read -field=value secret/ops/token/github', returnStdout: true)
                    }
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
                PREVIEW_VERSION = "0.0.0-SNAPSHOT-${TAG}"qq
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
                    sh "mvn -q -B install"
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
                    sh "mvn -q -B release:prepare"
                }
            }
        }
        stage('Release [ x.x ]') {
            when {
                expression { BRANCH_NAME ==~ /[0-9]\.[0-9]/ }
            }
            environment {
                TAG = 'stable'
                ORG = 'sidohaakma'
                GROUP_NAME = 'org.molgenis'
                APP_NAME = 'molgenis-app-maven-test'
                GITHUB_CRED = credentials('molgenis-jenkins-github-secret')
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
//                    sh "git remote set-url origin https://${env.GITHUB_CRED_PSW}@github.com/${ORG}/${APP_NAME}.git"
//                    sh "git checkout -f ${BRANCH_NAME}"
//                    sh ".release/generate_release_properties.bash ${APP_NAME} ${GROUP_NAME} ${env.RELEASE_SCOPE}"
                    sh "mvn release:perform"
//                    sh "git push --tags origin ${BRANCH_NAME}"
                }
            }
        }
    }
}
