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

    options {
        // Supprime le workspace automatiquement avant chaque build
        skipDefaultCheckout(false)
    }

    stages {

        stage('Checkout') {
            steps {
                sh "rm -rf *"
                sh "git clone https://github.com/vveame/workshop2-groupe1.git"
            }
        }

        stage('Build + Test + OWASP') {
            steps {
                dir('workshop2-groupe1/maven') {
                    // OWASP runs automatically because it's bound to verify
                    sh 'mvn clean verify'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'workshop2-groupe1/maven/target/dependency-check-report/**',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('workshop2-groupe1/maven') {

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
