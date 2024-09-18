pipeline {

     agent {
          // label 'docker-agent-alpine'
          label 'ssh-agent'
         }
     
     options {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '1'))
        timeout(time: 30, unit: 'MINUTES')
     }

    environment {
        // pendiente sonarq
        ANALYSIS_SONARQUBE = "true"
        ANALYSIS_OWASP = "false"
        SONARQUBE_SCANNER_HOME = tool 'sonarqube'
        PROJECT_NAME = "rga_tasks"
        PROJECT_CLIENT = "rga"
        SONARQUBE_TAG="1.0"
        SONARQUBE_CONFIG_FILE_PATH = "./sonar-project.properties"
    }

    tools{
         maven 'maven_3.9.9'
        }    
    
    stages {

        stage('Checkout') {
                steps {
                    git credentialsId: 'git-credentials', poll: false, 
                    url: 'https://github.com/argos-iot/simple-java-maven-app.git'
                }
        }

        stage('Sonarqube') {

            steps {
              withSonarQubeEnv(credentialsId: 'sonar-jenkins', installationName: 'sonarqube') {
                  // sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner -Dproject.settings=${env.SONARQUBE_CONFIG_FILE_PATH}"
		        sh "mvn clean verify sonar:sonar -Dsonar.projectKey=simple-mvn-test -Dsonar.java.binaries=target/classes"
              	}	
            }
	    }
   
        stage('Checking Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qualityGate = waitForQualityGate()
                        if (qualityGate.status != 'OK') {
                            error "El análisis de Sonar ha fallado con el estado: ${qualityGate.status}"
                        }
                    }
                }
            }
        }

	    stage('OWASP Dependency-Check Vulnerabilities') {
            when {
                 environment name: 'ANALYSIS_OWASP', value: 'true'
            }
              steps {
                dependencyCheck additionalArguments: ''' 
                            -o './'
                            -s './'
                            -f 'ALL' 
                            --prettyPrint''', odcInstallation: 'dependency-check'
                
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
             }
        }
	    
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    // Publicar el reporte de las pruebas unitarias usando JUnit
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
	    
        stage('Packaging') { 
            steps {
                sh "echo start building with mvn, skipping test"
		        // sh "mvn -DskipTests=true -Denforcer.skip=true clean package"
            }
            post {
                    success {
                        archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: false
                    }
                }
        }


    post {

        always {
            script {
                // Generar el changelog basado en los últimos 10 commits
                def changeLogText = ""
                def changeSet = currentBuild.changeSets

                for (int i = 0; i < changeSet.size(); i++) {
                    def entries = changeSet[i].items
                    for (int j = 0; j < entries.length && j < 10; j++) {
                        def entry = entries[j]
                        changeLogText += "Commit ${j+1}:\n"
                        changeLogText += "Autor: ${entry.author}\n"
                        changeLogText += "Mensaje: ${entry.msg}\n"
                        changeLogText += "Fecha: ${entry.timestamp}\n"
                        changeLogText += "---------------------------------------------\n"
                    }
                }

                // Guardar el changelog en un archivo
                writeFile file: 'changelog.txt', text: changeLogText
                archiveArtifacts artifacts: 'changelog.txt', allowEmptyArchive: false
            }
        }
	    
        success {
            archiveArtifacts artifacts: '**/*.jar,**/*.war,target/surefire-reports/*.xml',
                   allowEmptyArchive: true,
                   fingerprint: true,
                   onlyIfSuccessful: true
          }  
    }
}
