pipeline {
    agent any

    tools {
          jdk 'OpenJDK-17.0.7-LTS-AArch46'
          maven 'Maven-3.9.2'
          docker 'docker'
    }

    stages {
        stage('Build') {
            steps {
                sh "mvn -Dmaven.test.failure.ignore=true clean package"
            }

            post {
                success {
                    junit '**/target/surefire-reports/TEST-*.xml'
                    archiveArtifacts 'target/*.jar'
                }
            }
        }

        stage('Docker') {
            agent {
                label 'master'
            }
            steps {
                sh "docker build . -t springtest"
            }
        }

        stage('SonarQube') {
            steps {
                withSonarQubeEnv('Calypso Binar SonarQube Server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
    }
}
