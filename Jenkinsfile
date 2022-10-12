pipeline{
    agent any
    environment{
imageName = "nava9594/$JOB_NAME:v1.$BUILD_ID"
}
    tools{
        maven 'maven'
    }
    stages{
        stage('checkout the code'){
            steps{
                slackSend channel: 'hello-world', message: 'job started'
                git url:'https://github.com/NavnathChaudhari/devops-sec', branch: 'master'
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
                    withSonarQubeEnv(credentialsId: 'jenkins-sonar-token') {
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
        stage('Deploy application on k8s'){
            steps{
                sh 'sed -i "s#replace#${imageName}#g" deployment.yaml'
                sh 'kubectl apply -f deployment.yaml'
        }
        }
    }
        post{
        always{
          junit 'target/surefire-reports/*.xml'
          jacoco execPattern: 'target/jacoco.exec'
          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
          dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
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
