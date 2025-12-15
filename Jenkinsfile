pipeline {
    agent {
        docker {
            image 'my-maven-git-sonarscanner:latest'
            args '-v /var/jenkins_home/.m2:/var/jenkins_home/.m2'
        }
    }

    environment {
        SONAR_PROJECT_KEY = "DevopsPipeLine"
        SONAR_PROJECT_NAME = "java-maven"
    }

    stages {

        stage('Checkout') {
            steps {
                sh "rm -rf *"
                sh "git clone https://github.com/vveame/java-maven.git"
            }
        }

        stage('Build') {
            steps {
                dir('java-maven/maven') {
                    sh 'mvn clean test package'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('java-maven/maven') {

                    // Run sonar with Jenkins credentials + server
                    withSonarQubeEnv('java_maven') {
                        sh """
                        sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                            -Dsonar.projectVersion=1.0 \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.token=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }
    }
}
