library identifier: 'jenkins-core-library@kubernetes/1.0.0', retriever: modernSCM(
  [$class: 'GitSCMSource',
    remote: 'https://github.com/pdrodavi/jenkins-core-library.git'
  ])

pipeline {
    
  agent {
    kubernetes {
      yamlFile 'docker-maven-jnlp-pod-template.yaml'
    }
  }
  
  tools {
    maven "M3"
  }  
  
  stages {

    stage('Checkout') {
      steps {
        container('maven') {
          script {
            def lst = [];

            withCredentials([string(name: 'CREDGH', credentialsId: 'github-rest-token', variable: 'GITHUBRESTJWT')]) {
              
                inputName = input([
                        message: 'Input Name Repository',
                        parameters: [
                                string(name: 'Only Name')
                        ]
                ])

                httpRequest consoleLogResponseBody: true, customHeaders: [[maskValue: false, name: 'Accept', value: 'application/vnd.github+json'], [maskValue: false, name: 'Authorization', value: "Bearer ${GITHUBRESTJWT}"], [maskValue: false, name: 'X-GitHub-Api-Version', value: '2022-11-28']], outputFile: 'branches.json', url: "https://api.github.com/repos/pdrodavi/${inputName}/branches", wrapAsMultipart: false

                def props = readJSON file: "${env.WORKSPACE}/branches.json", returnPojo: true
                props.each { key, value ->
                    lst.add("$key.name")
                }

                inputBranch = input([
                        message: 'Choose Branch',
                        parameters: [
                                choice(name: 'Branches', choices: lst, description: 'Select branch for pipeline')
                        ]
                ])

                //deleteDir()
                println("Repositorio: https://github.com/pdrodavi/${inputName}.git")
                println("Branch selecionada: ${inputBranch}")
                git branch: "${inputBranch}", changelog: false, poll: false, url: 'https://pdrodavi:' + "${GITHUBRESTJWT}" + '@github.com/pdrodavi/' + "${inputName}" + '.git'
            }
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        container('docker') {
          println("Criando a imagem Docker")
          sh "docker build -t pdrodavi/${readJSON(file: 'package.json').name}:latest ."
        }
      }
    }
    
    stage('Publish Image') {
      steps {
        container('docker') {
          script {

            //withCredentials([string(name: 'CREDREGDHPD', credentialsId: 'docker-hub-pdrodavi-pass', variable: 'DOCKERHUBPDRODAVI')]) {
  
              inputPublish = input([
                      message: 'Publish to Registry?',
                      parameters: [
                              choice(name: 'Publish', choices: ['Yes', 'No'], description: 'Publish image to artifactory')
                      ]
              ])

              Boolean executeStage = false

              if ("${inputPublish}" == 'Yes') {
                  executeStage = true
              }

              conditionalStage("Publish Image", executeStage) {
                  //sh 'docker login -u pdrodavi -p ${DOCKERHUBPDRODAVI}'
                  sh 'docker login -u pdrodavi -p Docker@2022'
                  sh "docker push pdrodavi/${readJSON(file: 'package.json').name}:latest ."
              }


          //}
          
        }
      }
    }
    
  }

  }

  post {
      always {
        container('docker') {
          sh 'docker logout'
        }
      }
  }

}
