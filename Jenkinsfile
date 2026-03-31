pipeline {
    agent {
        docker {
            image 'maven:3.9.11-eclipse-temurin-17'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile -B -ntp'
            }
        }
        stage('Junit-Test') {
            steps {
                sh 'mvn test -Dmaven.test.failure.ignore=true -B -ntp'
            }
            post {
                success {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Jacoco-Coverage') {
            steps {
                sh 'mvn jacoco:report -B -ntp'
            }
            post {
                success {
                    recordCoverage(tools: [[parser: 'JACOCO']])
                }
            }
        }
        stage('Package') {
            steps {
                sh 'mvn package -B -ntp -DskipTests'
            }
        }
        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonarqube'){
                    sh 'env | sort'
                    script {
                        if (env.CHANGE_ID) {
                            sh """
                                mvn sonar:sonar -B -ntp \
                                -Dsonar.pullrequest.key=${env.CHANGE_ID} \
                                -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} \
                                -Dsonar.pullrequest.base=${env.CHANGE_TARGET}
                            """
                        } else {
                            def branchName = GIT_BRANCH.replaceFirst('^origin/', '')
                            println "Branch name: ${branchName}"
                            sh "mvn sonar:sonar -B -ntp -Dsonar.branch.name=${branchName} -Dsonar.branch.target=${branchName}"
                        }
                    }
                }
            }
        }
        stage('Artifactory') {
            steps {
                script {

                    sh 'env | sort'
                    env.MAVEN_HOME = '/usr/share/maven'
                    def releaseRepo = 'spring-petclinic-rest-release'
                    def snapshotRepo = 'spring-petclinic-rest-snapshot'
                    def server = Artifactory.server 'artifactory'
                    
                    def pom = readMavenPom file: 'pom.xml'
                    println pom.groupId

                    def groupIdPath = pom.groupId.replaceAll("\\.", "/")
                    println groupIdPath

                    def uploadSpec = """
                        {
                            "files": [
                                {
                                    "pattern": "target/.*.jar",
                                    "target": "${releaseRepo}/${groupIdPath}/${pom.artifactId}/${pom.version}/",
                                    "regexp": "true",
                                    "props": "build.url=${RUN_DISPLAY_URL};build.user=${currentBuild.getBuildCauses()[0].userId}"
                                }
                            ]
                        }
                    """
                    server.upload spec: uploadSpec

                }
            }
        }
    }
    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
        cleanup {
            cleanWs()
        }
    }
}