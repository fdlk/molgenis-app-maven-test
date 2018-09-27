pipeline {
    agent {
        kubernetes {
            label('molgenis')
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
                        env.CODECOV_TOKEN = sh(script: 'vault read -field=value secret/ops/token/codecov', returnStdout: true)
                        env.GITHUB_USER = sh(script: 'vault read -field=username secret/ops/token/github', returnStdout: true)
                    }
                }
                dir('/home/jenkins/.m2'){
                    stash includes: 'settings.xml', name: 'maven-settings'
                }
                input(message: 'Do you want to continue?')
            }
        }
        stage('Build [ pull request ]') {
            when {
                changeRequest()
                beforeAgent true
            }
            environment {
                //PR-1234-231
                TAG = "PR-${CHANGE_ID}-${BUILD_NUMBER}"
                //0.0.0-SNAPSHOT-PR-1234-231
                PREVIEW_VERSION = "0.0.0-SNAPSHOT-${TAG}"
            }
            stages {
                stage('Build') {
                    steps {
                        container('maven') {
                            sh "mvn versions:set -DnewVersion=${PREVIEW_VERSION} -DgenerateBackupPoms=false"
                            sh "mvn clean install -Dmaven.test.redirectTestOutputToFile=true -DskipITs -Ddockerfile.tag=${TAG} -Ddockerfile.skip=false"
                        }
                    }
                }
            }
        }

        stage('Build [ master ]') {
            when {
                branch 'master'
                beforeAgent true
            }
            agent {
                kubernetes {
                    label('molgenis-it')
                }
            }
            stages {
                stage('Build') {
                    steps {
                        dir('/home/jenkins/.m2'){
                            unstash 'maven-settings'
                        }
                        container('maven') {
                            sh "mvn -q -B clean verify"
                            sh "echo 'docker tag+push registry/artifact:dev'"
                            sh "echo 'docker tag+push registry/artifact:dev-$BUILD_NUMBER'"
                        }
                    }
                }
                stage('Deploy') {
                    steps {
                        container('helm') {
                            sh "echo 'helm upgrade dev-$BUILD_NUMBER'"
                        }
                    }
                }
            }
        }

        stage('Build [ x.x ]') {
            when {
                expression { BRANCH_NAME ==~ /[0-9]\.[0-9]/ }
                beforeAgent true
            }
            agent {
                kubernetes {
                    label('molgenis-it')
                }
            }
            stages {
                stage('Build [ x.x ]') {
                    steps {
                        dir('/home/jenkins/.m2'){
                            unstash 'maven-settings'
                        }
                        container('maven') {
                            sh "mvn -q -B clean verify"
                        }
                    }
                }
                stage('Prepare Release') {
                    steps {
                        timeout(time: 10, unit: 'MINUTES') {
                            input(message: 'Prepare to release?')
                        }
                        container('maven') {
                            sh "mvn -q -B release:prepare"
                            script {
                                env.RELEASE_TAG = sh(
                                        script: "grep project.rel release.properties | cut -d'=' -f2",
                                        returnStdout: true
                                )
                            }
                            sh "echo 'docker tag+push registry/artifact:$RELEASE_TAG'"
                        }
                    }
                }
                stage('Deploy on test') {
                    steps {
                        sh "echo 'helm upgrade test registry/artifact:$RELEASE_TAG'"
                    }
                }
                stage('Perform release') {
                    steps {
                        timeout(time: 10, unit: 'DAYS') {
                            input(message: 'Do you want to release?')
                        }
                        container('vault') {
                            script {
                                env.PGP_SECRETKEY = "keyfile:/home/jenkins/key.asc"
                                sh(script: 'vault read -field=secret.asc secret/ops/certificate/pgp/molgenis-ci > /home/jenkins/key.asc')
                            }
                        }
                        container('maven') {
                            sh "mvn -B release:perform"
                            sh "echo 'docker tag+push hub/artifact:$RELEASE_TAG'"
                            sh "echo 'docker tag+push hub/artifact:stable'"
                            sh "echo 'docker tag+push hub/artifact:${BRANCH_NAME}-stable'"
                        }
                    }
                }
            }
        }
    }
}