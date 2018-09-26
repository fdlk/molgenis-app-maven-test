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
                        sh "mkdir /home/jenkins/.m2"
                        sh(script: 'vault read -field=value secret/ops/jenkins/maven/settings.xml > /home/jenkins/.m2/settings.xml')
                        env.SONAR_TOKEN = sh(script: 'vault read -field=value secret/ops/token/sonar', returnStdout: true)
                        env.GITHUB_TOKEN = sh(script: 'vault read -field=value secret/ops/token/github', returnStdout: true)
                        env.PGP_PASSPHRASE = 'literal:' + sh(script: 'vault read -field=passphrase secret/ops/certificate/pgp/molgenis-ci', returnStdout: true)
                        env.PGP_SECRETKEY = "keyfile:/home/jenkins/key.asc"
                        sh(script: 'vault read -field=secret.asc secret/ops/certificate/pgp/molgenis-ci > /home/jenkins/key.asc')
                        env.CODECOV_TOKEN = sh(script: 'vault read -field=value secret/ops/token/codecov', returnStdout: true)
                        env.GITHUB_USER = sh(script: 'vault read -field=username secret/ops/token/github', returnStdout: true)
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
            steps {
                container('maven') {
                    sh "mvn -q -B clean verify"
                    sh "echo 'docker tag+push registry/artifact:latest'"
                }
            }
        }

        stage('Build [ x.x ]') {
            when {
                expression { BRANCH_NAME ==~ /[0-9]\.[0-9]/ }
            }
            stages {
                stage('Build'){
                    steps {
                        container('maven'){
                            sh "mvn -q -B clean verify"
                        }
                    }
                }
                stage('Prepare Release'){
                    steps {
                        timeout(time: 10, unit: 'MINUTES') {
                            input(message: 'Prepare to release?')
                        }
                        container('maven') {
                            sh "mvn -q -B release:prepare"
                            sh "echo 'docker tag+push registry/artifact:7.0.3'"
                        }
                    }
                }
                stage('Install on test') {
                    steps {
                        sh "echo 'helm upgrade test registry/artifact:7.0.3'"
                    }
                }
                stage('Perform release'){
                    steps {
                        timeout(time: 10, unit: 'DAYS') {
                            input(message: 'Do you want to release?')
                        }
                        container('maven'){
                            sh "mvn -B release:perform"
                            sh "echo 'docker tag+push hub/artifact:7.0.3'"
                            sh "echo 'docker tag+push hub/artifact:stable'"
                            sh "echo 'docker tag+push hub/artifact:${BRANCH_NAME}-stable'"
                        }
                    }
                }
            }
        }
    }
}
