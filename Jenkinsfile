#!groovy

@Library('plivo_standard_libs@production') _

deliveryPipeline ([
    disableUnitTests: true,
    disableLint: true,
    disableSonar: true,
    build: true
])

#!groovy

buildContext = [:]

buildContext.dockerFile = 'Dockerfile'
buildContext.projectDir = '.'

pipeline {
	agent {
		label 'c5_2xlarge_ubuntu_20_04_spot'
	}

	options {
		buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
		timeout(time: 1, unit: 'HOURS')
		timestamps()
	}

	stages {
		stage('Checkout') {
			steps {
				deleteDir()
                cleanTmp()
                sh 'rm -rfv * ||:'
                script {
                    reportDurationToCloudWatch {
                        scmExtensions = [
                            [
                                $class: 'SubmoduleOption',
                                disableSubmodules: false,
                                parentCredentials: true,
                                recursiveSubmodules: true,
                                reference: '',
                                trackingSubmodules: false
                            ]
                        ]

                        debugLog(scm.dump())
                        checkout([
                            $class: 'GitSCM',
                            branches: scm.branches,
                            doGenerateSubmoduleConfigurations: false,
                            extensions: scmExtensions,
                            submoduleCfg: [],
                            userRemoteConfigs: scm.userRemoteConfigs
                        ])
                        configDir = buildContext.projectDir + '/ci'
                        buildContext.configFile = "${configDir}/config.yml"

                        String version = new java.text.SimpleDateFormat("yy.MM.dd.${currentBuild.id}").format(new Date())
                        buildContext.version = version

                        debugCmd('ls -la *')

                        assert buildContext.configFile

                        debugLog('buildContext.configFile: ' + buildContext.configFile)
                        debugLog('buildContext.dockerFile: ' + buildContext.dockerFile)

                        projectConfig = readYaml file: buildContext.configFile
                        buildContext << projectConfig

                        buildContainer = "plivo/jenkins-ci/debian:jessie"
                        assert buildContainer != null

                        cleanTmp()

                        debugCmd('ls -l ||:')
                        debugCmd('ls -l ' + checkoutDir + '/ ||:')
                        debugCmd('ls -l ' + checkoutDir + '/*/ ||:')
                    }
                }
			}
		}

        stage('Build/Publish') {
            agent { 
                docker {
                    image buildContainer
                    reuseNode true
                }
            }

            steps {
                script {
                    reportDurationToCloudWatch {
                        build([
                            buildContext: buildContext
                        ])
                    }
                }
            }

            post {
                failure {
                    script { failedStage = env.STAGE_NAME }
                }
            }
        }

        stage('Build/Publish Docker Image') {
            steps {
                script {
                    reportDurationToCloudWatch {
                        dir("opensearch-operator") {
                            buildDockerImage([
                                buildContext: buildContext
                            ])
                        }
                        
                    }
                }
            }

            post {
                failure {
                    script { failedStage = env.STAGE_NAME }
                }
            }
        }
    }

	post {
		always {
			deleteDir()
		}
	}
}
