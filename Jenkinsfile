pipeline {
  agent any

  stages {

        stage('Build') {
            steps {
                withMaven(maven: 'MAVEN_ENV') {
                    sh "mvn clean install -DskipTests=true"
                }
            }
        }

 }
}  
