// Maven Shipyard Template v1.2.0
// ------------------------------------------------------------------------------------------------------------------------------

// Embeddable badge configuration. Under normal circumstances there is no need to change these values
def shipyardBuildBadge = addEmbeddableBadgeConfiguration(id: "shipyard-build", subject: "Shipyard Build")

pipeline {
    agent {
        node {
            label 'build'
        }
    }
// Location for setting global environment variables
    environment {
        EMAIL_RECIPIANTS = 'jcoleman@cynerge.com'
        NEXUS_USER = credentials('nexus-user') // pre-built into Shipyard
        NEXUS_PASS = credentials('nexus-pass') // pre-built into Shipyard
        STATUS_SUCCESS = ''
        //JENKINS_URL = "${JENKINS_URL}"
        JOB_NAME = "${JOB_NAME}"
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_PROJECT = 'hello-world-java'
        SONAR_REPORTS = 'target/surefire-reports'
        JACOCO_REPORT = "$WORKSPACE/tests/target/site/jacoco-aggregate/jacoco.xml"
        PA11Y_SOURCE = './**/src/**/**/**.html'
        NEXUS_RAW_REPO = credentials('nexus-raw-repo')
    }

    stages {
// Clear any previously compiled files, then compile and package artifact
        stage('Build') {
            // Use Maven container to run pipeline stages
            agent {
                docker {
                    image 'maven:3.6.2-jdk-13'
                }
            }
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
// Run any defined unit tests
        stage('Unit Testing') {
            // Use Maven container to run pipeline stages
            agent {
                docker {
                    image 'maven:3.6.2-jdk-13'
                }
            }
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit "java_webapp*/target/surefire-reports/*.xml"
                }
            }
        }
// Test app accessability by scanning HTML files
        stage('Pa11y') {
            agent {
                docker {
                    image 'cynergeconsulting/browser-node-12'
                    alwaysPull true
                    args '-u root'
                }
            }

            steps {
                sh 'rm -rf test-results || echo "directory does not exist"'
                sh 'mkdir test-results'
                sh 'chmod -R 777 test-results/'
                sh "pa11y-ci -T 5 ${env.PA11Y_SOURCE} --json > test-results/pa11y-ci-results.json"
                dir('test-results') {
                    sh 'pa11y-ci-reporter-html'
                }
            }
// Send test results to Nexus repository
            post {
                success {
                    // Do NOT delete the empty line underneath below curl command. It is necessary for script logic
                    dir('test-results') {
                        sh "curl -v --user '${NEXUS_USER}:${NEXUS_PASS}' --upload-file \"{\$(echo *.html | tr ' ' ',')}\" ${NEXUS_RAW_REPO}/Pa11y/${JOB_NAME}/${BRANCH_NAME}/${BUILD_NUMBER}/"

                    }
                }
            }
        }
// Trigger an integrity scan from Sonarqube's server. The detailed results will be available in your Sonarqube dashboard
        stage('Sonarqube Analysis') {
            // Use Maven container to run pipeline stages
            agent {
                node {
                    label 'master'
                }
            }
            environment {
                scannerHome = tool 'cynerge-sonarqube'
            }
            steps {
                withSonarQubeEnv('Cynerge Sonarqube') {
                    sh "mvn -Dsonar.login=$SONAR_TOKEN \
                     -Dsonar.projectKey=$SONAR_PROJECT \
                     -Dsonar.junit.reportPaths=$SONAR_REPORTS \
                     -Dsonar.coverage.jacoco.xmlReportPaths=$JACOCO_REPORT \
                     clean verify sonar:sonar"
                }
            }
        }
// Store your build artifact in a Nexus Repository. The delivery script will largely differ with app type
        stage('Store Maven Artifact') {
            // Use Maven container to run pipeline stages
            agent {
                docker {
                    image 'maven:3.6.2-jdk-13'
                }
            }
            environment { 
                MAVEN_USER = credentials('nexus-user')
                MAVEN_PASS = credentials('nexus-pass')
                MAVEN_REPO = credentials('nexus-maven-repo')

            }
            steps {
                sh 'env'
                sh "curl -v -u $MAVEN_USER:$MAVEN_PASS --upload-file java_webapp/target/java-webapp*.jar $MAVEN_REPO"
            }
        }
    }
    post {
// Send an email containing build status and other helpful info to appropriate recipiants
        success {
            script {
                env.STATUS_SUCCESS = 'Job Complete!'
                env.JENKINS_URL = "${JENKINS_URL}"
                env.JOB_NAME = "${JOB_NAME}"
                env.BUILD_STATUS = 'Success'
            sh 'printenv'
            emailext body: '''${SCRIPT, template="email_report.template"}''',
            mimeType: 'text/html',
            subject: 'Build # $BUILD_NUMBER - $BUILD_STATUS!',
            to: "${EMAIL_RECIPIANTS}"

            }
        }
        failure {
            script {
                env.STATUS_SUCCESS = 'Job Complete!'
                env.JENKINS_URL = "${JENKINS_URL}"
                env.JOB_NAME = "${JOB_NAME}"
                env.BUILD_STATUS = 'Failure'
            sh 'printenv'

            emailext body: '''${SCRIPT, template="email_report.template"}''',
            mimeType: 'text/html',
            subject: 'Build # $BUILD_NUMBER - $BUILD_STATUS!',
            to: "${EMAIL_RECIPIANTS}"

            }
        }
    
        cleanup {
// Clean up workspace after build
            cleanWs()
// Update build badge with passing or failing status
            script {
                shipyardBuildBadge.setStatus('running')
                try {
                    shipyardBuildBadge.setStatus('passing')
                } 
                catch (Exception err) {
                    shipyardBuildBadge.setStatus('failing')
                    shipyardBuildBadge.setColor('red')
                    error 'Build failed'
                }
            }
        }
    }
}