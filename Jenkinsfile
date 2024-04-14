pipeline {
  agent any
    environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "zhumazia/numeric-app:${GIT_COMMIT}"
    applicationURL="http://devsecops-zumi.eastus.cloudapp.azure.com/"
    applicationURI="/increment/99"
  }





  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' 
            }
        }   

    

      stage('Unit Test- JUnit and Jacoco') {
            steps {
              sh "mvn test"
            }
        }

      stage('Mutation Test -PIT') {
        steps {
          sh "mvn org.pitest:pitest-maven:mutationCoverage"
          }
      
      }
      stage('Sonarqube - SAST') {
        steps {
           withSonarQubeEnv('SonarQube') {
          sh " mvn sonar:sonar \
              -Dsonar.projectKey=numeric-appliction \
              -Dsonar.projectName='numeric-appliction' \
              -Dsonar.host.url=http://devsecops-zumi.eastus.cloudapp.azure.com:9000"
            }

           timeout(time: 2, unit: 'MINUTES') {
            script {
                waitForQualityGate abortPipeline: true
             }
           }
        }
      }

      stage('Vulnerability Scan -Docker'){
        steps {
              parallel(
                  "Dependency Scan": {
                    sh "mvn dependency-check:check"
                  },

                  "Trivy Scan": {
                    sh "bash trivy-docker-image-scan.sh"
                  },
                  "OPA Scan": {
                    sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
                  }
                )
        }
      }

       stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
              sh 'printenv'
              sh 'sudo docker build -t zhumazia/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push zhumazia/numeric-app:""$GIT_COMMIT""'
            }
          }
        }
        stage('Vulnerability Scan -K8S'){
          steps{
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          }
        }
       stage('Kubernetes deployment - DEV') {
          steps{
            parallel(
              "Deployment":{
                withKubeConfig([credentialsId:'kubeconfig']){
                  sh "sed -i 's#replace#zhumazia/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
                  sh "kubectl apply -f k8s_deployment_service.yaml"
                }
              },
              "Rollout Status":{
                withKubeConfig([credentialsId:'kubeconfig']){
                  sh "bash "

              }

            )
                }
        }
       
      }

    post { 
        always { 
            junit 'target/surefire-reports/*.xml'
            jacoco execPattern: 'target/jacoco.exec'
            pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
        }

      }
}