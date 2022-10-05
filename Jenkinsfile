pipeline {
  agent any

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar'  
            }
        }   
      stage('Unit Tests') {
            steps {
              sh "mvn test"
            }
            post{
              always{
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec' //to review the results of the unit tests. plugin added to pom.xml
              }
            }
        }
      stage('Docker Build and Push') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub', url: '']) {
                    sh 'printenv'
                    sh 'docker build -t rtravass/numeric-app-new:""$GIT_COMMIT"" .'
                    sh 'docker push rtravass/numeric-app-new:""$GIT_COMMIT""'
                }        
            }
        }     
    }
}
