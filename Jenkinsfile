@Library('devops-jenkins-lib') _

// Cancel previous builds if they're running
def buildNumber = env.BUILD_NUMBER as int
if (buildNumber > 1) milestone(buildNumber - 1)
milestone(buildNumber)

// Repository cache name
def REPO = "3d-hotspot-conversion.git"

// Set the tag name if available
if (env.TAG_NAME != null) {
    env.VERSIONING_GIT_TAG = "${env.TAG_NAME}"
}

// Set the branch name
if (env.CHANGE_BRANCH != null) {
    env.VERSIONING_GIT_BRANCH = env.CHANGE_BRANCH
} else {
    env.VERSIONING_GIT_BRANCH = env.BRANCH_NAME
}

pipeline {
    agent {
        kubernetes {
            yamlFile "extra-containers.yaml"
        }
    }
    stages {
        stage('Application') {
            stages {
                stage('Compile') {
                    steps {
                        container("builder") {
                            withMaven(options: [artifactsPublisher(disabled: true)], globalMavenSettingsConfig: 'devops') {
                                sh "mvn package jar:test-jar install:install -DskipTests"
                            }
                        }
                    }
                }
                stage('Test') {
                    steps {
                        container("builder") {
                            withMaven(options: [artifactsPublisher(disabled: true)], globalMavenSettingsConfig: 'devops') {
                                sh "mvn test -fae"
                            }
                        }
                    }
                }
                stage('Verify') {
                    steps {
                        container("builder") {
                            withMaven(options: [artifactsPublisher(disabled: true)], globalMavenSettingsConfig: 'devops') {
                                sh 'mvn verify -DskipTests -fae'
                            }
                        }
                    }
                }
                stage('Static Analysis') {
                    // Due to how SonarQube works, we need to analyze the builds differently depending on kind
                    parallel {
                        stage('Mainline and Branches') {
                            when {
                                not {
                                    anyOf {
                                        changeRequest()
                                        buildingTag()
                                    }
                                }
                            }
                            steps {
                                container("builder") {
                                    withMaven(options: [artifactsPublisher(disabled: true)], globalMavenSettingsConfig: 'devops') {
                                        withSonarQubeEnv('Devops Sonarqube') {
                                            sh "mvn initialize sonar:sonar -Dsonar.branch.name=${env.BRANCH_NAME}"
                                        }
                                        timeout(time: 5, unit: 'MINUTES') {
                                            waitForQualityGate abortPipeline: true
                                        }
                                    }
                                }
                            }
                        }

                        stage('Pull Requests') {
                            when {
                                allOf{
                                    changeRequest()
                                    not {
                                        // Don't analyze the merged build as it's ephemeral
                                        branch '*-merge'
                                    }
                                }
                            }
                            steps {
                                container("builder") {
                                    withMaven(options: [artifactsPublisher(disabled: true)], globalMavenSettingsConfig: 'devops') {
                                        withSonarQubeEnv('Devops Sonarqube') {
                                            sh "mvn initialize -Psonar sonar:sonar -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} -Dsonar.pullrequest.key=${env.CHANGE_ID} -Dsonar.pullrequest.base=${env.CHANGE_TARGET}"
                                        }
                                        timeout(time: 5, unit: 'MINUTES') {
                                            waitForQualityGate abortPipeline: true
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
                stage('Deploy') {
                    // when { anyOf { branch 'master'; branch 'release/*'; branch 'reint/*' } }
                    steps {
                        container("builder") {
                            withMaven(options: [artifactsPublisher(disabled: true)], globalMavenSettingsConfig: 'devops') {
                                sh "mvn jar:test-jar package deploy -DskipTests -Dskip.tests=true -Dmaven.install.skip=true"
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        failure {
            script {
                email.failure ""
            }
        }
        fixed {
            script {
                email.fixed ""
            }
        }
    }
}