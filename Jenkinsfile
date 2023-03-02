pipeline{
    agent any
    environment{
      imageName = "nava9594/$JOB_NAME:v1.$BUILD_ID"
      deploymentName = "devsecops"
      containerName = "devsecops-container"
      serviceName = "devsecops-svc"
      applicationURL="192.168.122.25"
      applicationURI="/increment/99"
}
    tools{
        maven 'maven'
    }
    stages{
        stage('checkout the code'){
            steps{
                slackSend channel: 'hello-world', message: 'job started'
                git url:'https://github.com/Rushi017/devops-sec', branch: 'master'
            }
        }
        stage('build the code'){
            steps{
                sh 'mvn clean package'
            }
        }
        stage('Unit Tests - JUnit and Jacoco') {
      steps {
        sh "mvn test"
      }
    }
         stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
    }
    
        stage("sonar quality check"){
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-server') {
                            sh "mvn sonar:sonar -f /var/lib/jenkins/workspace/spring-boot-pipeline/pom.xml"
                    }
                    timeout(time: 1, unit: 'HOURS') {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }
                } 
            }
        }
        stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
          "Dependency Scan": {
            sh "mvn dependency-check:check"
          },
          "Trivy Scan": {
            sh "bash trivy-docker-image-scan.sh"
              sh "chown -R jenkins:jenkins trivy"
              sh "chmod -R 777 /var/lib/jenkins/workspace/spring-boot-pipeline/trivy/db/metadata.json"
          },
          "OPA Conftest": {
            sh "/usr/local/bin/conftest test --policy opa-docker-security.rego Dockerfile"
          }
        )
      }
    }
        stage('create docker image'){
            steps{
                sh '''docker image build -t $JOB_NAME:v1.$BUILD_ID .
docker image tag $JOB_NAME:v1.$BUILD_ID nava9594/$JOB_NAME:v1.$BUILD_ID
docker image tag $JOB_NAME:v1.$BUILD_ID nava9594/$JOB_NAME:latest'''
            }

        }
        stage('push the image into docker hub'){
            steps{
                withCredentials([string(credentialsId: 'Docker pass', variable: 'docker_pass')]) {
                    sh "docker login -u nava9594 -p ${docker_pass}"

}
                    sh '''docker image push nava9594/$JOB_NAME:v1.$BUILD_ID
docker image push nava9594/$JOB_NAME:latest 
docker image rmi $JOB_NAME:v1.$BUILD_ID nava9594/$JOB_NAME:v1.$BUILD_ID nava9594/$JOB_NAME:latest'''

            }
        }
        stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
          "OPA Scan": {
            sh '/usr/local/bin/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "bash kubesec-scan.sh"
          },
         "Trivy Scan": {
           sh "bash trivy-k8s-scan.sh"
         }
        )
      }
    }
        stage('K8S Deployment - DEV') {
       steps {
         parallel(
           "Deployment": {
            sh "sed -i 's#replace#${imageName}#g' k8s_deployment_service.yaml"
            sh "kubectl apply -f k8s_deployment_service.yaml"
          },
           "Rollout Status": {
            sh "bash k8s-deployment-rollout-status.sh"
           }
         )
     }
    }
        stage('Integration Tests - DEV') {
    steps {
    script {
     try {
       withKubeConfig([credentialsId: 'kubeconfig-2']) {
          sh "bash integration-test.sh"
        }
       } catch (e) {
        withKubeConfig([credentialsId: 'kubeconfig-2']) {
          sh "kubectl -n default rollout undo deploy ${deploymentName}"
        }
        throw e
      }
    }
  }
}
stage('OWASP ZAP - DAST') {
       steps {
         withKubeConfig([credentialsId: 'kubeconfig-2']) {
           sh 'bash zap.sh'
        }
       }
   }
stage('Prompte to PROD?') {
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

     stage('K8S Deployment - PROD') {
       steps {
         parallel(
           "Deployment": {
             withKubeConfig([credentialsId: 'kubeconfig-2']) {
               sh "sed -i 's#replace#${imageName}#g' k8s_PROD-deployment_service.yaml"
               sh "kubectl -n prod apply -f k8s_PROD-deployment_service.yaml"
             }
           },
           "Rollout Status": {
             withKubeConfig([credentialsId: 'kubeconfig-2']) {
               sh "bash k8s-PROD-deployment-rollout-status.sh"
             }
           }
         )
       }
     }

     stage('Integration Tests - PROD') {
       steps {
         script {
           try {
             withKubeConfig([credentialsId: 'kubeconfig-2']) {
               sh "bash integration-test-PROD.sh"
             }
           } catch (e) {
             withKubeConfig([credentialsId: 'kubeconfig-2']) {
               sh "kubectl -n prod rollout undo deploy ${deploymentName}"
             }
            throw e
           }
         }
       }
     }
}
         
        post{
        always{
          junit 'target/surefire-reports/*.xml'
          jacoco execPattern: 'target/jacoco.exec'
          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
          dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
          publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML Report', reportTitles: 'OWASP ZAP HTML Report'])
        }
        success{
            echo "========pipeline executed successfully ========"
            slackSend channel: 'hello-world', message: ' job success'
        }
        failure{
            echo "========pipeline execution failed========"
            slackSend channel: 'hello-world', message: ' job failed'
        }
        }
    }
