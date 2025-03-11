pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'mysonar'
    }
    stages {
        stage ("Clean") {
            steps {
                cleanWs()
            }
        }
        stage ("Code") {
            steps {
                git branch: 'main', url: 'https://github.com/charan-kilana/K8s_Project.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage ("Quality Gates") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage ("Install dependencies") {
            steps {
                sh 'npm install'
            }
        }
        stage ("OWASP") {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'Dp-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ("Trivy scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage ("Build Dockerfile") {
            steps {
                sh 'docker build -t image1 .'
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker-password') {
                        sh "docker tag image1 charansuryakilana/Zomato:mydockerimage"
                        sh "docker push charansuryakilana/Zomato:mydockerimage"
                    }
                }
            }
        }
        stage ("Scan image") {
            steps {
                sh 'trivy image charansuryakilana/Zomato:mydockerimage'
            }
        }
      
        }
    }
}
