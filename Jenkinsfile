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

                httpRequest consoleLogResponseBody: true, customHeaders: [[maskValue: false, name: 'Accept', value: 'application/vnd.github+json'], [maskValue: false, name: 'Authorization', value: "Bearer ${GITHUBRESTJWT}"], [maskValue: false, name: 'X-GitHub-Api-Version', value: '2022-11-28']], outputFile: 'branches.json', url: "https://api.github.com/repos/pdrodavi/ivalid-api/branches", wrapAsMultipart: false

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

                deleteDir()
                println("Branch selecionada: ${inputBranch}")
                git branch: "${inputBranch}", changelog: false, poll: false, url: 'https://pdrodavi:' + "${GITHUBRESTJWT}" + '@github.com/pdrodavi/ivalid-api.git'
            }
          }
        }
      }
    }
      
    stage('Analysis') {
      steps {
          script {
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
          println("Criando a imagem Docker")
          sh "docker build -t pdrodavi/${readMavenPom().getArtifactId()}:latest ."
        }
      }
    }
    
    stage('Publish Image') {
      steps {
          script {
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
    
  }
}
