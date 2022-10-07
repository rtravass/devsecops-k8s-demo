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
      stage('Mutation Tests - PIT') {
            steps {
              sh "mvn org.pitest:pitest-maven:mutationCoverage"
            }
            post{
              always{
                pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml' 
              }
            }
      }

      stage('SonarQube - SAST') {
            steps {
              sh "mvn clean verify sonar:sonar \
                -Dsonar.projectKey=numeric-application \
                -Dsonar.host.url=http://devsecops.centralindia.cloudapp.azure.com:9000 \
                -Dsonar.login=sqp_d81e9d7cd900909f49cc7a7ecf3dbe26a1bf48bf"
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
       stage('Kubernetes Deployment - Dev') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "sed -i 's#replace#rtravass/numeric-app-new:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
                    sh "kubectl apply -f k8s_deployment_service.yaml"
              }        
            }
        }       
    }
}
