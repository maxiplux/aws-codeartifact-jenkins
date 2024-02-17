#!/usr/bin/env groovy

pipeline {
    agent any

	environment {
        RELEASE_BRANCH = 'main'
    }

    parameters {

      validatingString(
            name: 'RELEASE_VERSION',
            regex: /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-((?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+([0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$/,
            failedValidationMessage: 'The version format is not valid',
            description: 'The release version to build (format: X.Y.Z)'
        )

      validatingString(
            name: 'DEVELOPMENT_VERSION',
            regex: /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-((?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+([0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$/,
            failedValidationMessage: 'The version format is not valid',
            description: 'The next version that the pom will be set to once the build has been completed. (format : X.Y.Z-SNAPSHOT)'
        )

    }

    options {
        timestamps()
        disableConcurrentBuilds()
		// Timeout counter starts AFTER agent is allocated
        timeout(time: 30, unit: 'MINUTES')
		// Keep the 10 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }


    stages {
        stage('Checkout code') {
            steps {
                git branch: "${RELEASE_BRANCH}" , url: 'https://github.com/anicetkeric/aws-codeartifact-jenkins.git'
                }
        }

	    stage('Check existing tag') {
             when {
                expression {
                    RELEASE_TAG = sh (script: 'git tag -l $RELEASE_VERSION',returnStdout: true).trim()
                    return RELEASE_TAG == params.RELEASE_VERSION
                }
            }
            steps {
				echo(">> Tag $RELEASE_VERSION already exists")

                sh 'git tag -d $RELEASE_VERSION'
            }
        }

        stage("Release setup") {
            steps {
				echo ">> RELEASE_VERSION: $params.RELEASE_VERSION"

				echo ">> Version update"

				withMaven(maven: 'MAVEN_ENV') {
					sh 'mvn versions:set -DnewVersion=$RELEASE_VERSION -DprocessAllModules -DgenerateBackupPoms=false'
                }

				echo ">> Commit the modified POM file and tag the release"
                sh('''
                    git config user.name 'aek'
                    git config user.email 'anicetkeric@outlook.com'
                    git add :/*pom.xml
                    git commit -m "Release $RELEASE_VERSION"
                    git tag -a $RELEASE_VERSION -m "New Tag $RELEASE_VERSION"

				''')

                echo ">> Release setup successfully"
            }
        }

        stage("Release Build and deploy") {
            steps {

				withMaven(maven: 'MAVEN_ENV') {
					sh "mvn clean install -DskipTests=true"
                }

				echo ">> Publish tag to repository"
                configFileProvider([configFile(fileId: '1e855f66-f777-4538-9d9a-782c61054866', variable: 'MyGlobalSettings')]) {

                    withAWS(credentials: 'AWS_IAM_CREDENTIALS', region: 'us-east-1') {
                        script {
                            env.CODEARTIFACT_AUTH_TOKEN = sh (script: 'aws codeartifact get-authorization-token --domain boottech --domain-owner $AWS_ACCOUNT_ID --region us-east-2 --query authorizationToken --output text',returnStdout: true).trim()
                        }

                        withMaven(maven: 'MAVEN_ENV') {
                            sh "mvn -s $MyGlobalSettings clean deploy -DskipTests=true"
                        }

                    }
                }
            }
        }

		stage("Adding next version") {
            steps {

				echo ">> DEVELOPMENT_VERSION: $DEVELOPMENT_VERSION"

				withMaven(maven: 'MAVEN_ENV') {
                    sh "mvn versions:set -DnewVersion=$DEVELOPMENT_VERSION -DprocessAllModules -DgenerateBackupPoms=false"
                }

				echo ">> Commit the modified POM file and push next version"
				withCredentials([gitUsernamePassword(credentialsId: 'GITHUB_TOKEN', gitToolName: 'Default')]) {
                   sh('''
					git add :/*pom.xml
                    git commit -m "Prepare the next snapshot version : $DEVELOPMENT_VERSION"
					git push origin $RELEASE_BRANCH

					git push origin refs/tags/$RELEASE_VERSION

					''')
                }

				 echo ">> The next snapshot version pushed successfully"
            }
        }



    }
}
