pipeline {
  agent any 
  tools {
    maven 'maven'
  }
  environment {
  DOCKER_IMAGE_NAME = "ProjectName"
  REPO_NAME = "RepoName"
  }

  stages {
    stage ('Initialize') {
      steps {
        sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
            ''' 
        }
      }
    stage ('Unit Test') {
      steps {
          sh 'mvn test'
      }   
    }
     stage ('Trivy Check Git Secrets') {
      steps {
        sh ' trivy repo --format "json" -o "trivy-git-repo-scan.json"  https://github.com/RepoName'
       
      }
     }
   
     stage ('Dependency Check') {
      steps {
         sh 'rm owasp* || true'
         sh 'wget "https://raw.githubusercontent.com/MarwenSoula/javulna/master/owasp-dependency-check.sh" '
         sh 'chmod +x owasp-dependency-check.sh'
         sh 'bash owasp-dependency-check.sh'
           
      }
    }
    
    stage ('Build Maven') {
      steps {
          sh 'mvn clean package'    
       }
    }
    stage ('SAST SonarQube') {
        steps {
           withSonarQubeEnv('sonar') {
             sh 'mvn clean package sonar:sonar '
        }
      }
    }
    stage ('SonarQube Quality Gate') {
        steps {
          script{
           waitForQualityGate abortPipeline: false, credentialsId: 'sonar-api' 
        }
      }   
    }
    stage ('Upload Artifact Nexus') {
      steps {
        script{
         def readPomVersion = readMavenPom file: 'pom.xml'
         nexusArtifactUploader artifacts: 
           [
             [artifactId: 
              'javulna',
              classifier: '',
              file: 'target/ArtifactName.war',
              type: 'war']
           ], 
           credentialsId: 'nexus',
           groupId: 'com.kalavit',
           nexusUrl: 'IP@:8081',
           nexusVersion: 'nexus3',
           protocol: 'http',
           repository: 'java-pipeline',
           version: "${readPomVersion.version}" 
      }
      }
    }
   stage ('Docker Build & Tag Image') {
     steps {
       script{
         def readPomVersion = readMavenPom file: 'pom.xml'
         sh "docker rmi ${REPO_NAME}/${DOCKER_IMAGE_NAME}:${readPomVersion.version} || true "
         sh "docker  build . -t ${REPO_NAME}/${DOCKER_IMAGE_NAME}:${readPomVersion.version} "
       
      }
     }
   }
    stage ('Scan Docker Image Trivy') {
      steps {
        script{
         def readPomVersion = readMavenPom file: 'pom.xml'
          sh " trivy image --format json -o trivy-docker-image-scan.json ${REPO_NAME}/${DOCKER_IMAGE_NAME}:${readPomVersion.version} "
          
      }
    }
    }
    stage ('DockerHub Push Image') {
     steps {
       script{
         withCredentials([string(credentialsId: 'DockerHubPassword', variable: 'DockerHub_Pwd')]) {
           sh "docker login --username username  --password ${DockerHub_Pwd} " 
         }
         def readPomVersion = readMavenPom file: 'pom.xml'
         sh "docker  push  ${REPO_NAME}/${DOCKER_IMAGE_NAME}:${readPomVersion.version} "
       
      }
     }
   }
    stage ('Ansibe Docker Deploy') {
      steps {
        sh "ansible-playbook -v deploy-container.yaml "      
       }
    }
  
    stage ('DAST ZAP') {
      steps {
         sh 'docker run -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py -t http://IP@/:8080/javulna/ -g gen.conf -x ./ZAP-report.xml '
      }
    }
   stage ('Ansible Prepare Security Scan Reports') {
      steps {
         sh 'ansible-playbook  defectdojo-import.yaml '
      }
    }
   stage ('Ansible-DefectDojo Import Reports') {
      steps {
         sh ' ansible-playbook Run-defectdojo.yaml -v '
      }
   }
  }
}
