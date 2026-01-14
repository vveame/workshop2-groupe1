pipeline {
    agent none
    
    parameters {
        booleanParam(name: 'RUN_OWASP', defaultValue: false, description: 'Run OWASP Dependency Check?')
    }

    environment {
        SONAR_PROJECT_KEY  = "DevopsPipeLine"
        SONAR_PROJECT_NAME = "java-maven"
        IMAGE_NAME = "my-maven-git-sonarscanner:latest"
    }

    stages {
        
        stage('Trivy Scan') {
            agent any
            steps {
                script {
                    sh "docker image inspect ${IMAGE_NAME}"

                    echo "Scanning CRITICAL vulnerabilities (build must fail)"
                    def critical = sh(
                        script: """
                        docker run --rm \
                          -v /var/run/docker.sock:/var/run/docker.sock \
                          trivy-cached:latest image \
                          --severity CRITICAL \
                          --exit-code 1 \
                          --skip-db-update \
                          --scanners vuln \
                          ${IMAGE_NAME}
                        """,
                        returnStatus: true
                    )

                    if (critical != 0) {
                        error "CRITICAL vulnerabilities detected – build aborted"
                    }

                    echo "Scanning MEDIUM vulnerabilities (build becomes UNSTABLE)"
                    def medium = sh(
                        script: """
                        docker run --rm \
                          -v /var/run/docker.sock:/var/run/docker.sock \
                          trivy-cached:latest image \
                          --severity MEDIUM \
                          --exit-code 1 \
                          --skip-db-update \
                          --scanners vuln \
                          ${IMAGE_NAME}
                        """,
                        returnStatus: true
                    )

                    if (medium != 0) {
                        echo "MEDIUM vulnerabilities detected – marking build as UNSTABLE"
                        currentBuild.result = 'UNSTABLE'
                    }

                    echo "LOW vulnerabilities are allowed"

                    echo "Generating full Trivy report"
                    sh 'mkdir -p reports'
                    
                    sh """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v \$(pwd)/reports:/reports \
                      trivy-cached:latest image \
                      --format json \
                      --skip-db-update \
                      --scanners vuln \
                      ${IMAGE_NAME} > reports/trivy-report.json
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "reports/trivy-report.json",  allowEmptyArchive: true
                }
            }
        }

        stage('Checkout') {
            agent {
                docker {
                    image IMAGE_NAME
                    args '-v /var/jenkins_home/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                deleteDir()
                sh 'git clone https://github.com/vveame/workshop3-groupe1.git'
            }
        }

        stage('Build & Test') {
            agent {
                docker {
                    image IMAGE_NAME
                    args '-v /var/jenkins_home/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                dir('workshop3-groupe1/maven') {
                    sh 'mvn clean compile test'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'workshop3-groupe1/maven/target/surefire-reports/*.xml',
                                     allowEmptyArchive: true
                    junit allowEmptyResults: true, testResults: 'workshop3-groupe1/maven/target/surefire-reports/*.xml'
                }
            }
        }

        stage('OWASP Dependency Check') {
            when {
                expression {
                    echo "RUN_OWASP param is: ${params.RUN_OWASP}"
                    return params.RUN_OWASP == true
                }
            }
            agent {
                docker {
                    image IMAGE_NAME
                    args '-v /var/jenkins_home/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                dir('workshop3-groupe1/maven') {
                    sh 'mvn org.owasp:dependency-check-maven:check'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'workshop3-groupe1/maven/target/dependency-check-report.html',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            agent {
                docker {
                    image IMAGE_NAME
                    args '-v /var/jenkins_home/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                dir('workshop3-groupe1/maven') {

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

    post {
        success  { echo "Pipeline completed successfully" }
        unstable { echo "Pipeline completed with warnings" }
        failure  { echo "Pipeline failed" }
    }
}
