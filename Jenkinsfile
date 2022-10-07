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
            // post{
            //   always{
            //     junit 'target/surefire-reports/*.xml'
            //     jacoco execPattern: 'target/jacoco.exec' //to review the results of the unit tests. plugin added to pom.xml
            //   }
            // }
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
              withSonarQubeEnv('SonarQube'){
                  sh "mvn clean verify sonar:sonar \
                    -Dsonar.projectKey=numeric-application \
                    -Dsonar.host.url=http://devsecops.centralindia.cloudapp.azure.com:9000"
              }
              timeout(time: 2, unit: 'MINUTES') {
                script {
                  waitForQualityGate abortPipeline: true
                }
              }
           }
      }   
      // stage('Vulnerability Scan') {    added to the scan step mentioned below
      //       steps {
      //               sh 'mvn dependency-check:check'
      //       }
      //       // post{
      //       //   always{
      //       //     //publish the html report
      //       //     dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      //       //   }
      //       // }        
      // } 

      stage('Vulnerability Scan - Docker') {
          steps {
              parallel(
                "Dependency Scan": {
                    sh "mvn dependency-check:check"
                },
                "Trivy Scan":{
                    sh "bash trivy-docker-image-scan.sh"
                },
                "OPA Conftest":{
                    sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
                }  
              )
         }
      }

      stage('Docker Build and Push') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub', url: '']) {
                    sh 'printenv'
                    sh 'sudo docker build -t rtravass/numeric-app-new:""$GIT_COMMIT"" .'
                    sh 'docker push rtravass/numeric-app-new:""$GIT_COMMIT""'
                    // sh 'sudo docker build -t rtravass/numeric-app-new:latest .'
                    // sh 'docker push rtravass/numeric-app-new:latest'
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
    post { 
        always { 
            junit 'target/surefire-reports/*.xml'
            jacoco execPattern: 'target/jacoco.exec' //to review the results of the unit tests. plugin added to pom.xml
            //pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml' //publish the html report
        }
    }
}
