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
            sh 'ls -a'
            sh 'pwd'
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
                sh 'ls -a'
                sh 'pwd'
            }
          }
        }
      }
    }
      
    stage('Analysis') {
      steps {
          script {
            sh 'ls -a'
            sh 'pwd'
            inputAnalysis = input([
                    message: 'Analysis SonarQube?',
                    parameters: [
                            choice(name: 'Analysis', choices: ['Yes', 'No'], description: 'Run on specific analysis')
                    ]
            ])

            Boolean executeStage = false

            if ("${inputAnalysis}" == 'Yes') {
                executeStage = true
            }

            conditionalStage("Analysis", executeStage) {

                if ("${inputAnalysis}" == 'Yes') {
                    withSonarQubeEnv('sonarqube') {
                        sh "mvn -B clean verify sonar:sonar"
                    }
                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        cleanWs()
                        error "Pipeline aborted due to quality gate failure: ${qualitygate.status}"
                    }
                } else {
                    println("Step Skipped")
                }
            }
          }
      }
    }  

    stage('Package') {
      steps {
          script {
            sh 'ls -a'
            sh 'pwd'
            println("Realizando construção do artefato")
            println("Artifact: " + readMavenPom().getArtifactId())
            println("Version: " + readMavenPom().getVersion())
            sh "mvn -Dmaven.test.skip=true -Dmaven.test.failure.ignore clean package"
          }
      }
    }

    stage('Build Image') {
      steps {
        container('docker') {
          sh 'ls -a'
          sh 'pwd'
          println("Criando a imagem Docker")
          sh "docker build -t pdrodavi/${readMavenPom().getArtifactId()}:latest ."
        }
      }
    }
    
    stage('Publish Image') {
      steps {
          script {
            sh 'ls -a'
            sh 'pwd'
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
                withDockerRegistry(credentialsId: Constants.JENKINS_JFROG_CREDENTIALS_ID, url: Constants.JENKINS_JFROG_URL_REGISTRY) {
                    sh "docker push pdrodavi/${readMavenPom().getArtifactId()}:latest"
                }
            }
          }
      }
    }

    stage('Deployment') {
      steps {
        container('docker') {
          println("Executando Deploy")
          httpRequest consoleLogResponseBody: true, outputFile: 'deployment.yaml', url: "https://raw.githubusercontent.com/pdrodavi/pipeline-kubernetes-jenkins/main/deployment.yaml", wrapAsMultipart: false
          sh 'ls -a'
          sh 'pwd'
          sh 'cat deployment.yaml | sed "s/{{NAME_IMAGE}}/${readMavenPom().getArtifactId()}/g" | kubectl apply -f -'
          //sh "docker build -t pdrodavi/${readMavenPom().getArtifactId()}:latest ."
        }
      }
    }
    
  }
}
