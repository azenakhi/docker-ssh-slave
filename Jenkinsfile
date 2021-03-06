/* NOTE: this Pipeline mainly aims at catching mistakes (wrongly formed Dockerfile, etc.)
 * This Pipeline is *not* used for actual image publishing.
 * This is currently handled through Automated Builds using standard Docker Hub feature
*/
pipeline {
    agent none

    options {
        buildDiscarder(logRotator(daysToKeepStr: '10'))
        timestamps()
    }

    triggers {
        pollSCM('H/24 * * * *') // once a day in case some hooks are missed
    }

    stages {
        stage('Build Docker Image') {
            parallel {
                stage('Windows') {
                    agent {
                        label "windock"
                    }
                    options {
                        timeout(time: 60, unit: 'MINUTES')
                    }
                    environment {
                        DOCKERHUB_ORGANISATION = "${infra.isTrusted() ? 'jenkins' : 'jenkins4eval'}"
                    }
                    steps {
                        powershell '& ./make.ps1 test'
                        script {
                            def branchName = "${env.BRANCH_NAME}"
                            if (branchName ==~ 'master') {
                                // we can't use dockerhub builds for windows
                                // so we publish here
                                infra.withDockerCredentials {
                                    powershell '& ./make.ps1 publish'
                                }
                            }

                            if (env.TAG_NAME != null) {
                                def tagItems = env.TAG_NAME.split('-')
                                if(tagItems.length == 2) {
                                    // we need to build and publish the tag version
                                    infra.withDockerCredentials {
                                        powershell "& ./make.ps1 -PushVersions -Tag ${env.TAG_NAME} publish"
                                    }
                                }
                            }
                        }

                        powershell '& docker system prune --force --all'
                    }
                }
                stage('Linux') {
                    agent {
                        label "docker&&linux"
                    }
                    options {
                        timeout(time: 30, unit: 'MINUTES')
                    }
                    steps {
                        script {
                            if(!infra.isTrusted()) {
                                deleteDir()
                                checkout scm
                                sh '''
                                make build
                                make test
                                docker system prune --force --all
                                '''
                            }
                        }
                    }
                }
            }
        }
    }
}

// vim: ft=groovy
