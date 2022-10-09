pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "rtravass/numeric-app-new:${GIT_COMMIT}"
    //applicationURL="http://devsecops-demo.eastus.cloudapp.azure.com"
    applicationURL="http://devsecops.centralindia.cloudapp.azure.com"
    applicationURI="/increment/99"
  }

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

      // stage('Vulnerability Scan - Kubernetes'){    //Updated below
      //       steps{
      //           sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
      //       }
      // }

      stage('Vulnerability Scan - Kubernetes') {
            steps {
                parallel(
                    "OPA Scan": {
                      sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
                    },
                    "Kubesec Scan": {
                      sh "bash kubesec-scan.sh"
                    },
                    // "Trivy Scan": {
                    //   sh "bash trivy-k8s-scan.sh"
                    // }
                )
            }
      }

      // stage('Kubernetes Deployment - Dev') {      //updated stage to verify k8s deployment status
      //       steps {
      //           withKubeConfig([credentialsId: 'kubeconfig']) {
      //               sh "sed -i 's#replace#rtravass/numeric-app-new:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
      //               sh "kubectl apply -f k8s_deployment_service.yaml"
      //         }        
      //       }
      //   }       

      stage('K8S Deployment - DEV') {
          steps {
            parallel(
              "Deployment": {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                  sh "bash k8s-deployment.sh"
                }
              },
              "Rollout Status": {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                  sh "bash k8s-deployment-rollout-status.sh"
                }
              }
            )
          }
      }

      // stage('Integration Tests - DEV') {
      //     steps {
      //       script {
      //         try {
      //           withKubeConfig([credentialsId: 'kubeconfig']) {
      //             sh "bash integration-test.sh"
      //           }
      //         } catch (e) {
      //           withKubeConfig([credentialsId: 'kubeconfig']) {
      //             sh "kubectl -n default rollout undo deploy ${deploymentName}"
      //           }
      //           throw e
      //         }
      //       }
      //     }
      // }

      stage('OWASP ZAP - DAST') {
          steps {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh 'bash zap.sh'
            }
          }
      }

      stage('Promote to PROD?') {
          steps {
            timeout(time: 2, unit: 'DAYS') {
              input 'Do you want to Approve the Deployment to Production Environment/Namespace?'
            }
          }
      }

      stage('K8S CIS Benchmark') {
            steps {
              script {

                parallel(
                  "Master": {
                    sh "bash cis-master.sh"
                  },
                  "Etcd": {
                    sh "bash cis-etcd.sh"
                  },
                  "Kubelet": {
                    sh "bash cis-kubelet.sh"
                  }
                )

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
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML REPORT', reportTitles: 'OWASP ZAP HTML REPORT', useWrapperFileDirectly: true])    //publish OWASP DAST html report
        }
    }
}
