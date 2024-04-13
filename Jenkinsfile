pipeline {
  agent any

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
            post {
             always {
              junit 'target/surefire-reports/*.xml'
               jacoco execPattern: 'target/jacoco.exec'
              }
             }
        }

      stage('Mutation Test -PIT') {
        steps {
          sh "mvn org.pitest:pitest-maven:mutationCoverage"
          }
        post {
          always {
            pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
          }
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
              script{
              waitForQualityGate abortPipeline: true
              }
          }
        }
      }
     

       stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
              sh 'printenv'
              sh 'docker build -t zhumazia/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push zhumazia/numeric-app:""$GIT_COMMIT""'
            }
          }
       }
       stage('Kubernetes deployment - DEV') {
                steps{
                    withKubeConfig([credentialsId:'kubeconfig']){
                      sh "sed -i 's#replace#zhumazia/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
                      sh "kubectl apply -f k8s_deployment_service.yaml"
                    }
                }
       }
       
    }
}